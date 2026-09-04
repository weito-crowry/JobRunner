# 03. Runtime / Scheduling 詳細設計

- Status: Draft v2.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `05`, `06`, `08`, `09`, `10`

## 1. 基本原則

1. Runnerが取得する単位はconcrete internal Job。
2. RunnerはWorkflow Runを占有しない。
3. 同一Workflow Run internal Job同時実行最大1。
4. 別Workflow RunはRunner Pool空きまで並列。
5. External/Human/Reusable待ちはRunner非占有。
6. Dynamic templateは仮想groupでRunner/Attempt対象ではない。
7. DB永続状態がSource of Truth。
8. claim/state transitionはconditional transaction。
9. Priority変更はpreemptしない。
10. Job result自動再利用は同一Workflow Run内だけ。
11. Lease/Retry/Retention等の期限処理はMaintenance Loop。
12. Concurrency scopeは`workflow_id + group`で、別Workflow IDの同名groupは競合しない。
13. Concurrency waiting orderの時刻はRun作成時刻ではなく、実際にwaiterになった`concurrency_queued_at`を使う。
14. Workflow Run `started_at`はrow作成時刻ではなく、**初めてConcurrency admissionされ`running`になった時刻**を表す。

## 2. Runtime起動

Parent Process正規順:

1. Core config/data root/Integration Bootstrap entrypoint解決
2. `Integration Bootstrap(role=parent)`
3. Registry/Auth/Secrets/optional Store factory構築
4. SQLite/PayloadStore/ArtifactStore初期化
5. Migration
6. Workflow Resolver
7. Runner Pool config validation
8. `runtime_instance_id`発行
9. non-terminal Recovery
10. Runner Pool Supervisor
11. Maintenance Loop
12. Scheduling受付

Bootstrap/Provider/Storage/Migration/Pool validation失敗時はScheduling開始しない。

## 3. Root Workflow Run start

Start前にDefinition/Input/Registry version/Runner Pool/Expression/Reusable/Concurrency/Authorizationを検証。

Root Run start時にcurrent System workflow defaultsを`01` shapeでsnapshotし、そこからeffective settings/Retentionを算出する。

Initial Run priority:

```text
wf_start.priority specified -> request value
otherwise -> Workflow Definition priority
```

Start transaction:

- Run Definition/Input snapshot
- `system_workflow_defaults_json`
- effective settings/Retention
- Action/Validator version snapshot
- initial priority
- concurrency snapshot/admission
- static non-dynamic Job rows
- Event
- optional idempotency result

Dynamic templateはDefinition snapshotに存在する仮想group。Run startでは`job_runs` row無し。

Static Jobは最初`status=queued, ready_at=NULL`。

Workflow Run initial state/timestamps:

- `created_at` = Workflow Run rowを作成したcanonical UTC時刻。以後不変
- Concurrencyなし、またはslot admission成功 -> `status=running, wait_reason=NULL, concurrency_queued_at=NULL, started_at=created_at`（同じstart transaction時刻）
- `on-limit=queue`でslot不足 -> `status=queued, wait_reason=concurrency, concurrency_queued_at=created_at, started_at=NULL`
- `on-limit=reject`でslot不足 -> Run rowを作らずreject

MVPのWorkflow Run `queued` は**Concurrency admission待ち専用**とする。通常のJob dependency/Runner待ちはWorkflow Run自体を`queued`へ戻さず`running`のまま。

Queued Runが後でslotを取得したtransactionで、`started_at IS NULL`ならそのadmission時刻を1回だけ設定する。同一Runでその後Pause/Resume/Manual Retryしても`started_at`を書き換えない。

`workflow_started` Eventは`started_at`が初めて設定されるadmission transactionでexactly once記録する。Concurrency待ちとしてrowを作っただけでは`workflow_started`を発行せず、必要に応じ`workflow_queued`等のstate Eventを記録してよい。

Concurrencyは`08`の `(workflow_id, concurrency_group)` scopeで判定する。

## 4. Child Workflow Run start

Reusable Childは`06` bindingから:

- Definition
- Action/Validator versions
- inherited System workflow defaults
- Child effective settings/Retention
- Actor/AccessScope/source_identity

をcopyする。Current System configを読み直さない。

Child Run priority=current root Workflow Run priority。

Nested Childも同じbaseline inheritanceを繰り返す。

Child自身のConcurrencyはChild `workflow_id + resolved group` scopeで`08` admissionを行う。Parentとgroup文字列が同じでもWorkflow IDが異なれば競合しない。

Child Run timestampsもRootと同じ意味:

- Child row作成時=`created_at`
- admission成功 -> `status=running, concurrency_queued_at=NULL, started_at=created_at`（同transaction）
- Queue時 -> `status=queued, wait_reason=concurrency, concurrency_queued_at=created_at, started_at=NULL`
- 後日slot取得 -> `started_at`をそのadmission時刻へ初回設定

## 5. Status / readiness / Workflow timestamp invariant

Workflow=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Canonical Workflow status invariant:

```text
queued    -> wait_reason=concurrency, concurrency_queued_at non-NULL, no concurrency slot
running   -> admitted non-terminal Run, wait_reason=NULL, concurrency_queued_at=NULL
paused    -> pause済みnon-terminal Run
completed -> conclusion non-NULL, concurrency_queued_at=NULL
```

Timestamp semantics:

```text
created_at   = Run row creation time; immutable
started_at   = first admission to running; set once; NULL if never admitted
completed_at = current terminal completion time; NULL while non-terminal
```

- `running` -> `started_at` non-NULL
- admitted `paused` -> `started_at` non-NULL
- concurrency waiter `queued/paused` -> `started_at` may remain NULL until first admission
- Runが一度もadmitされないままcancel/completedになった場合は`started_at=NULL`を許可
- Manual Retry reopenでは`started_at`を保持し、`completed_at`だけclearする

Paused Run:

- `wait_reason=concurrency`なら元Concurrency waiter、`concurrency_queued_at`保持、slot無し
- それ以外なら元admitted Run、`concurrency_queued_at=NULL`、slot保持

Concrete Job=`queued|running|waiting_external|waiting_review|waiting_child|completed`、Conclusion=`success|failure|cancelled|skipped|blocked`。

Dynamic group status/conclusionは`05`で導出。

Queued concrete Job:

- `ready_at=NULL`: dependency/condition activation未完了
- `ready_at!=NULL`: 次Attempt用persistent Input snapshot確定済み
- `retry_not_before>now`: Retry backoff待ち

`ready_at`はInternal/non-internal Retry activationの準備時刻であり、External Task claimのavailability時刻ではない。External claimは`external_tasks.available_at`を使う。

## 6. Concrete Job初回activation

前提:

1. non-terminal
2. Run `status=running`（Concurrency admitted）、cancel/pause中でない
3. dependencies準備済み
4. condition dependency set terminal
5. current contextでJob `if`評価

`if=false` -> Attempt無しで`completed/skipped`。

`if=true`:

1. continue-on-error snapshot
2. `with` -> persistent Input/bindings/digest
3. executor別activation

Internal:

- pending snapshotをJob row
- ready_at=now
- status queued
- Attempt無し

External:

- Attempt snapshot
- Task `available_at=now`
- Log
- waiting_external

Human:

- Attempt snapshot
- Review + Log
- waiting_review

Reusable:

- binding
- Attempt snapshot
- Child admission + Log
- waiting_childまたはChild start failure

Pre-Attempt Input/condition failureはJob-level failure。Input snapshot無しならManual Retry不可。

## 7. Dynamic template activation

Definition snapshotから未確定Dynamic templateを探索。

- Root: template + Run
- Nested: template + each parent generated Job

Run `status=running`かつDependencies terminal後`05`のif/foreach/key/order/atomic expansion。

Template自身のAttempt/Execution Log/Retry target無し。Expansion failureは`dynamic_expansions.failure_json` + Workflow failure。

## 8. Internal selection order

Internal Runner candidateは次のtotal order:

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. Job `ready_at` ASC
6. **Job `job_key` ASC**
7. Job Run ID ASC

Static Jobの`source_order`はMVPでは全て0でよく、YAML declaration order自体をScheduling保証にしない。同条件のStatic Jobはstableな`job_key`で順序を確定する。

Generated Jobは`order_rank/source_order`が`job_key`より先に効くため、Dynamic `order_by`/source array orderの意味は維持される。

Job Run IDはUUID由来のopaque IDでありcross-run再現順序を意味しない。同一DB状態内の最終tie-breakとしてのみ使う。

CandidateのWorkflow Runは`status=running`必須。Internalはresolved Pool exact一致。

## 9. External Task claim order

External TaskはJob `ready_at`ではなくTask `available_at`を使う。CandidateのWorkflow Runは`status=running`必須。

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. `external_tasks.available_at` ASC
6. **Job `job_key` ASC**
7. Job Run ID ASC
8. Task ID ASC

Lease expiry requeueではsame Taskの`available_at=expiry processing time`へ更新し、再claim順に反映する。

## 10. Root priority update

`wf_priority_update` はroot non-terminal Runだけ許可。

1 transactionで:

- root priority更新
- `root_workflow_run_id=root.id` の全non-terminal descendant Child Run priorityを同値へ更新
- Event/idempotency result

Current running Jobをpreemptしない。Future Childは更新後root priorityを継承。

## 11. Atomic internal claim

Candidate=`internal + queued + ready_at + due + pending snapshot + Workflow Run running + same Run running internal無し + Pool一致`。

1 transactionでJob running + new Attempt snapshot copy + Runner ownership + Log/Event。Attempt後pending clear。

## 12. Queued Retry activation for non-internal executor

External/Human/Reusable Retryは:

```text
Job status=queued
ready_at non-NULL
pending snapshot present
retry_not_before NULL or <=now
Workflow Run status=running
```

SchedulerはCancel gate後、`with`を再評価せずpending snapshot exact copyでnew Attempt。

- external -> new Task `available_at=activation time` / waiting_external
- human -> Review/waiting_review
- reusable -> same binding Child admission/waiting_childまたはstart failure

Attempt後pending clear。

## 13. Maintenance Loop

Workflow Scheduler/Cronではなく期限/housekeeping専用。

対象=`retry_not_before`, Lease expiry, concurrency wake, Retention, orphan cleanup, idempotency cleanup。

Nearest deadline + condition/event wait。Default max sleep5秒。

Due boundaryは`08` canonical timestampで`now >= deadline`。

Pause中Lease/Retention継続。Restart時overdue/cleanup/Retention先処理。

## 14. Concurrency admission / wake

Exact contract=`08 §23`。

- scope=`(workflow_id, group)`
- active holder countは同じscopeだけ
- candidate/active holdersの`max-runs`差はconservative minimum capacity
- `on-limit=queue`は`status=queued, wait_reason=concurrency, concurrency_queued_at=now`
- `on-limit=reject`はRun startならRun rowを作らずreject、completed Run Manual Retryなら既存Runを変更せずreject
- release/wakeは **priority DESC -> concurrency_queued_at ASC -> ID ASC** でhead-of-line fairness

Waiting candidateは`status=queued, wait_reason=concurrency, concurrency_queued_at non-NULL`だけ。Paused concurrency waiterはwake対象外だが、pause前の`concurrency_queued_at`を保持する。

Slot取得commit時:

```text
status=running
wait_reason=NULL
concurrency_queued_at=NULL
if started_at IS NULL: started_at=admission time
```

`started_at`を設定した場合は同transactionで`workflow_started` Eventをexactly once記録する。

Paused admitted holderはslotを保持する。Cancel/completionでholder解放。

## 15. Terminal後activation

Run `status=running`の範囲でIdempotentに:

- downstream concrete Job
- Dynamic group/expansion
- Reusable parent
- Result Reuse pending
- Workflow Output/conclusion
- concurrency release/wake
- Workflow progress再計算

## 16. Stored Result Reuse Context

Successful Attempt terminal時:

```text
workflow_run_id
job_key
definition_hash
input_digest
direct_upstream_artifacts
executor_identity
validator_identity
reuse_eligible
```

Direct Artifact identity=`artifact_id + optional digest`をdependency/name順canonicalize。

Executor identity:

- internal: Action ID/version
- external: `jobrunner.external_llm.v1`
- human: `jobrunner.human.v1`
- reusable:
  - Child Definition hash
  - Child Action versions
  - Child Validator versions
  - Child System workflow defaults JSON
  - Child effective settings JSON
  - Child Retention policy JSON

Validator identity=none null or ID/version。

PriorityはScheduling metadataでresult semanticsではないためreuse identityへ含めない。

## 17. Reuse eligibility

Successful Attemptは原則eligible。ただし以下を1回でも使ったAttemptは`reuse_eligible=false`:

- Runtime Handle `state.get`
- Runtime Handle `state.set`
- `secret_bindings` がnon-empty
- persistent Input/direct upstream Artifact以外のArtifactを`artifact.materialize`
- executor extensionが明示`reuse_eligible=false`

Secret valueは保存/hashしないためbinding付きAttemptの過去結果同一性を証明しない。

Expressionの`state`利用は、それが`with`ならInput digestへ入り、`if`なら§18でcurrent再評価されるため、それだけでineligibleにはしない。

## 18. Manual Retry後successful Job strict reuse

Current dependenciesから副作用無しで:

1. dependency terminal
2. current if=true
3. current with -> expected persistent Input/bindings/digest
4. digest一致
5. direct Artifact一致
6. Definition/executor/Validator identity一致
7. current Registry snapshot version exact
8. Payload/required Artifact integrity
9. eligible
10. stored Outputをexecutor固有の現在有効なSchema/Validator/success_if契約で再検証

Pass -> success維持 + `job_result_reused`。

Fail -> `successful_job_result_not_reusable`, retryable=false, new Workflow Run要求。Same Runでnew Input自動再実行しない。

## 19. Dynamic expansion reuse

Manual Retry後completed/success Dynamic groupはGenerated Job reuseより前に`05` expansion digest check。Exact matchのみ保持。Mismatch=`dynamic_expansion_not_reusable`。

Whole skipped groupのみ未実行として再評価可能。

## 20. Manual Retry Run reopen

Completed failure Runのreopenは`10`に従う。

- Concurrency scopeを再取得
- rejectならRun state無変更
- queueならpending Job Inputを保持したまま`status=queued, wait_reason=concurrency, concurrency_queued_at=retry request time`
- admittedなら`status=running, wait_reason=NULL, concurrency_queued_at=NULL`
- **どの場合も既存の`started_at`は書き換えない**
- reopen commit時にold Workflow Output current pointerをclear
- successful descendant reuse check/new Attemptはslot取得後の`running`状態で進める

Manual Retryでqueueへ入るRunは**元のRun `created_at`を待ち順に使わない**。新たにqueueへ入った時刻を使うため、古いRunが新規waiterへ不当に割り込まない。

## 21. Pause / Resume

Pauseはroot non-terminal Runに適用する。

### admitted Run (`running`)

- `status=paused`
- `wait_reason=NULL, concurrency_queued_at=NULL`
- `started_at`保持
- running internalは継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing submit/review/started Child進行可
- Lease/Retention継続
- Concurrency holder維持

### concurrency waiter (`queued, wait_reason=concurrency`)

- `status=paused`
- `wait_reason=concurrency` と元 `concurrency_queued_at`を保持
- `started_at`は未admitならNULLのまま
- slotは持たない
- concurrency wake candidateから除外
- Job/Dynamic activation無し

Resume:

- paused + `wait_reason=concurrency` -> `status=queued`へ戻す。**元の`concurrency_queued_at`を保持**してslot admission待ち
- paused admitted holder -> `status=running`へ戻し同slotで再開。`started_at`は保持
- ready_at non-NULL queued Jobはpending Input再評価無し

## 22. Cancel

queued/waiting cancel、Lease invalidate、running internal cancel、Child cancel、new activation禁止。Cancel後alwaysでもnew Job無し。

Cancel/result raceは`10`。Admitted holderはCancel/completionでConcurrency holder解放。Concurrency waiterはslot無し。Run completed化時`concurrency_queued_at=NULL`へclearする。

一度もadmitされていないConcurrency waiterをCancelしてcompleted/cancelledにした場合、`started_at=NULL`のままを正規状態とする。

## 23. Workflow conclusion

Success=required concrete Jobs/groups success、intentional skip、allowed continue-on-error、reuse pending無し。

Failure=non-allowed failure/blocked、Dynamic failure/not-reusable、Child failure、Workflow Output failure、successful Job not reusable、engine fatal。

Cancel requestが成立したRunは`cancelled`。

Terminal化時に`completed_at=current canonical UTC time`を設定する。Manual Retry reopen時だけ`completed_at`をclearし、次terminal化で新しい完了時刻を設定する。

## 24. Recovery

Restart:

- Bootstrap
- Workflow status/wait_reason/concurrency_queued_at/started_at invariant repair
- queued pending snapshots
- due noninternal Retry
- running internal recovery
- Lease/Review/Child
- Dynamic/reuse pending
- root/Child priority整合repair
- concurrency waiting/holder再計算
- Retention

Recoveryは正しい`concurrency_queued_at`と`started_at`を保持し、current wall clockへ書き換えない。Legacy/inconsistent row repair規則はmigration/schema version内で明示し、正常rowへ推測で新時刻を入れない。

Completed RunはRecoveryだけでreopenしない。

## 25. 受入条件

1. Root system baseline/effective setting snapshot
2. created_at=row creation immutable / started_at=first admission / completed_at=current terminal
3. initial admitted Run sets started_at / queued Run leaves NULL
4. later admission sets started_at exactly once + workflow_started Event
5. never-admitted cancelled Run keeps started_at NULL
6. Manual Retry reopen preserves started_at and clears/replaces completed_at only
7. admitted Run=running / concurrency waiter=queued only
8. concurrency waiter has queued_at / admitted has NULL
9. initial/Child queue sets concurrency_queued_at
10. paused holder vs paused waiter slot + queued_at/started_at semantics
11. resume waiter preserves original queue time
12. Manual Retry reopen queue gets new retry-time queue timestamp
13. Child inherits Parent Run baseline, not current System
14. root priority resolution/update/descendant propagation
15. Dynamic template no Job/Attempt
16. static ready_at NULL
17. internal pending/claim copy
18. all-executor Retry pending
19. Internal order uses ready_at / External order uses task available_at
20. static stable job_key tie-break / Dynamic order preserved
21. opaque ID final tie-break semantics
22. one-running/Pool
23. Maintenance due boundary
24. concurrency scope workflow_id+group
25. mixed max-runs/head-of-line fairness uses concurrency_queued_at
26. Manual Retry concurrency reacquire/output clear linkage
27. strict current if/Input/Artifact/version validation
28. Secret-bound/state/non-input Artifact reuse ineligible
29. Reusable identity includes Child baseline/settings/Retention
30. Dynamic expansion reuse
31. recovery idempotency/status/timestamp repair
