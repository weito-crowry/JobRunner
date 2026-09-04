# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.7
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

Workflow Run initial status:

- Concurrencyなし、またはslot admission成功 -> `status=running, wait_reason=NULL`
- `on-limit=queue`でslot不足 -> `status=queued, wait_reason=concurrency`
- `on-limit=reject`でslot不足 -> Run rowを作らずreject

MVPのWorkflow Run `queued` は**Concurrency admission待ち専用**とする。通常のJob dependency/Runner待ちはWorkflow Run自体を`queued`へ戻さず`running`のまま。

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

Child slot admission成功 -> Child `status=running`。Queue時のみChild `status=queued, wait_reason=concurrency`。

## 5. Status / readiness

Workflow=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Canonical Workflow status invariant:

```text
queued    -> wait_reason=concurrency, no concurrency slot
running   -> admitted non-terminal Run, concurrency設定時はslot holder
paused    -> pause済みnon-terminal Run
completed -> conclusion non-NULL
```

Paused Run:

- `wait_reason=concurrency`なら元Concurrency waiterでslot無し
- それ以外なら元admitted Runでslotを保持

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
6. Job Run ID ASC

Job Run IDはUUID由来のopaque IDでありcross-run再現順序を意味しない。同一DB状態内の最終tie-breakとしてのみ使う。

CandidateのWorkflow Runは`status=running`必須。Internalはresolved Pool exact一致。

## 9. External Task claim order

External TaskはJob `ready_at`ではなくTask `available_at`を使う。CandidateのWorkflow Runは`status=running`必須。

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. `external_tasks.available_at` ASC
6. Job Run ID ASC
7. Task ID ASC

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
- `on-limit=queue`は`status=queued, wait_reason=concurrency`
- `on-limit=reject`はRun startならRun rowを作らずreject、completed Run Manual Retryなら既存Runを変更せずreject
- release/wakeはpriority -> created_at -> ID順でhead-of-line fairness

Waiting candidateは`status=queued, wait_reason=concurrency`だけ。Paused concurrency waiterはwake対象外。

Slot取得commit時に`status=running, wait_reason=NULL`へ変更する。

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
- queueならpending Job Inputを保持したまま`status=queued, wait_reason=concurrency`
- admittedなら`status=running`
- reopen commit時にold Workflow Output current pointerをclear
- successful descendant reuse check/new Attemptはslot取得後の`running`状態で進める

## 21. Pause / Resume

Pauseはroot non-terminal Runに適用する。

### admitted Run (`running`)

- `status=paused`
- running internalは継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing submit/review/started Child進行可
- Lease/Retention継続
- Concurrency holder維持

### concurrency waiter (`queued, wait_reason=concurrency`)

- `status=paused`
- `wait_reason=concurrency`を保持
- slotは持たない
- concurrency wake candidateから除外
- Job/Dynamic activation無し

Resume:

- paused + `wait_reason=concurrency` -> `status=queued`へ戻してslot admission待ち
- paused admitted holder -> `status=running`へ戻し同slotで再開
- ready_at non-NULL queued Jobはpending Input再評価無し

## 22. Cancel

queued/waiting cancel、Lease invalidate、running internal cancel、Child cancel、new activation禁止。Cancel後alwaysでもnew Job無し。

Cancel/result raceは`10`。Admitted holderはCancel/completionでConcurrency holder解放。Concurrency waiterはslot無し。

## 23. Workflow conclusion

Success=required concrete Jobs/groups success、intentional skip、allowed continue-on-error、reuse pending無し。

Failure=non-allowed failure/blocked、Dynamic failure/not-reusable、Child failure、Workflow Output failure、successful Job not reusable、engine fatal。

Cancel requestが成立したRunは`cancelled`。

## 24. Recovery

Restart:

- Bootstrap
- Workflow status/wait_reason invariant repair
- queued pending snapshots
- due noninternal Retry
- running internal recovery
- Lease/Review/Child
- Dynamic/reuse pending
- root/Child priority整合repair
- concurrency waiting/holder再計算
- Retention

Completed RunはRecoveryだけでreopenしない。

## 25. 受入条件

1. Root system baseline/effective setting snapshot
2. admitted Run=running / concurrency waiter=queued only
3. paused holder vs paused waiter slot semantics
4. Child inherits Parent Run baseline, not current System
5. root priority resolution/update/descendant propagation
6. Dynamic template no Job/Attempt
7. static ready_at NULL
8. internal pending/claim copy
9. all-executor Retry pending
10. Internal order uses ready_at / External order uses task available_at
11. stable ID tie-break semantics
12. one-running/Pool
13. Maintenance due boundary
14. concurrency scope workflow_id+group
15. mixed max-runs/waiter fairness
16. Manual Retry concurrency reacquire/output clear linkage
17. strict current if/Input/Artifact/version validation
18. Secret-bound/state/non-input Artifact reuse ineligible
19. Reusable identity includes Child baseline/settings/Retention
20. Dynamic expansion reuse
21. recovery idempotency/status repair
