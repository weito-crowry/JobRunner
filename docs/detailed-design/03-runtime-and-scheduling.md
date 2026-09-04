# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.6
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

Concurrencyは`08`の `(workflow_id, concurrency_group)` scopeで判定する。

## 4. Child Workflow Run start

Reusable Childは`06` bindingからDefinition/versions/System baseline/settings/Retention/Actor/Scope/source_identityをcopyする。Current System configを読み直さない。

Child Run priority=current root Workflow Run priority。Nested Childも同じbaseline inheritance。

Child自身のConcurrencyはChild `workflow_id + resolved group` scopeで`08` admissionを行う。Parentとgroup文字列が同じでもWorkflow IDが異なれば競合しない。

## 5. Status / readiness

Workflow=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Concrete Job=`queued|running|waiting_external|waiting_review|waiting_child|completed`、Conclusion=`success|failure|cancelled|skipped|blocked`。

Dynamic group status/conclusionは`05`。

Queued concrete Job:

- `ready_at=NULL`: dependency/condition activation未完了
- `ready_at!=NULL`: 次Attempt用persistent Input snapshot確定済み
- `retry_not_before>now`: Retry backoff待ち

`ready_at`はInternal/non-internal Retry activation準備時刻。External claim availabilityは`external_tasks.available_at`。

## 6. Concrete Job初回activation

前提:

1. non-terminal
2. Run cancel/pause/concurrency waitでない
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

1. binding確定
2. Parent Attempt snapshot作成
3. `06` Child concurrency admission
4. admit/queue -> Child Run作成 + Log、Parent=`waiting_child`
5. reject -> Child Run無し、Parent Attempt/Job=`failure(child_workflow_start_failed)`

Pre-Attempt Input/condition failureはJob-level failure。Input snapshot無しならManual Retry不可。

## 7. Dynamic template activation

Definition snapshotから未確定Dynamic templateを探索。

- Root: template + Run
- Nested: template + each parent generated Job

Dependencies terminal後`05`のif/foreach/key/order/atomic expansion。

Template自身のAttempt/Execution Log/Retry target無し。Expansion failureは`dynamic_expansions.failure_json` + Workflow failure。

## 8. Internal selection order

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. Job `ready_at` ASC
6. Job Run ID ASC

Job Run IDは同一DB状態内の安定した最終tie-breakとしてのみ使う。Internalはresolved Pool exact一致。

## 9. External Task claim order

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. `external_tasks.available_at` ASC
6. Job Run ID ASC
7. Task ID ASC

Lease expiry requeueではsame Taskの`available_at=expiry processing time`へ更新する。

## 10. Root priority update

`wf_priority_update` はroot non-terminal Runだけ許可。

1 transactionでroot + 全non-terminal descendant Child Run priorityを同値へ更新しEvent/idempotency resultを保存。Preempt無し。Future Childは更新後root priorityを継承。

## 11. Atomic internal claim

Candidate=`internal + queued + ready_at + due + pending snapshot + same Run running internal無し + Pool一致`。

1 transactionでJob running + new Attempt snapshot copy + Runner ownership + Log/Event。Attempt後pending clear。

## 12. Queued Retry activation for non-internal executor

External/Human/Reusable Retry:

```text
status=queued
ready_at non-NULL
pending snapshot present
retry_not_before NULL or <=now
```

SchedulerはPause/Cancel/concurrency gate後、`with`を再評価せずpending snapshot exact copyでnew Attempt。

- external -> new Task `available_at=activation time` / waiting_external
- human -> Review/waiting_review
- reusable -> `06` Child concurrency admissionを再実行。admit/queueならChild/waiting_child、rejectならParent Attempt failure

Attempt後pending clear。

## 13. Maintenance Loop

対象=`retry_not_before`, Lease expiry, concurrency wake, Retention, orphan cleanup, idempotency cleanup。

Nearest deadline + condition/event wait。Default max sleep5秒。Due boundary=`now >= deadline`。

Pause中Lease/Retention継続。Restart時overdue/cleanup/Retention先処理。

## 14. Concurrency admission / wake

Exact contract=`08 §23`。

- scope=`(workflow_id, group)`
- active holder countは同じscopeだけ
- max-runs差はconservative minimum capacity
- queue=`status=queued, wait_reason=concurrency`
- reject=Run startならRun row無し、completed Run Manual Retryなら既存Run無変更
- release/wake=priority -> created_at -> ID順のhead-of-line fairness

Paused holderはslot維持。Cancel/completionで解放。

## 15. Terminal後activation

Idempotentに downstream concrete Job / Dynamic / Reusable parent / Result Reuse / Workflow Output+conclusion / concurrency release+wake / progress を進める。

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
- reusable: Child Definition hash + Action/Validator versions + System baseline + effective settings + Retention

PriorityはScheduling metadataなのでreuse identityへ含めない。

## 17. Reuse eligibility

Successful Attemptは原則eligible。ただし以下のいずれかで`reuse_eligible=false`。

- `secret_bindings_json` が空配列ではない
- Runtime Handle `state.get`
- Runtime Handle `state.set`
- persistent Input/direct upstream Artifact以外のArtifactを`artifact.materialize`
- executor extensionが明示`reuse_eligible=false`

### Secret bindingを不適格にする理由

Secret valueは意図的に永続化せず、`input_digest`にも含めない。したがって後のManual Retry時に「成功Attemptと現在のSecret実値が同一」とCoreは証明できない。

そのためSecret-backed Actionの過去成功結果は自動再利用しない。Target Retryでは同じbinding名/pathを使ってActionを再実行し、そのAttempt開始時のcurrent Secret valueをmaterializeする。

`state.set`も即時副作用を再現できないため自動reuseしない。

Expressionの`state`利用は、`with`ならInput digestへ入り、`if`なら§18でcurrent再評価されるため、それだけではineligibleにしない。

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
9. `reuse_eligible=true`
10. stored Outputをexecutor固有の現在有効なSchema/Validator/success_if契約で再検証

Pass -> success維持 + `job_result_reused`。

Fail -> `successful_job_result_not_reusable`, retryable=false, new Workflow Run要求。Same Runでnew Input自動再実行しない。

## 19. Dynamic expansion reuse

Manual Retry後completed/success Dynamic groupはGenerated Job reuseより前に`05` expansion digest check。Exact matchのみ保持。Mismatch=`dynamic_expansion_not_reusable`。

Whole skipped groupのみ未実行として再評価可能。

## 20. Manual Retry Run reopen

Completed failure Runのreopenは`10`。

- Concurrency scope再取得
- rejectならstate無変更
- queueならpending Input保持 + `wait_reason=concurrency`
- old Workflow Output current pointer clear
- new AttemptはConcurrency gateを越えるまで作らない

## 21. Pause / Resume

Pause:

- running internal継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing submit/review/started Child進行可
- Lease/Retention継続
- Concurrency holder維持

Resume:

- ready_at NULL concrete Job/Dynamicをcurrent dependencyから再評価
- ready_at non-NULL queued Jobはpending Input再評価無し
- concurrency wait中ならslot待ち

## 22. Cancel

queued/waiting cancel、Lease invalidate、running internal cancel、Child cancel、new activation禁止。Cancel後alwaysでもnew Job無し。

Cancel/result race=`10`。Cancel/completionでConcurrency holder解放。

## 23. Workflow conclusion

Success=required concrete Jobs/groups success、intentional skip、allowed continue-on-error、reuse pending無し。

Failure=non-allowed failure/blocked、Dynamic failure/not-reusable、Child failure、Workflow Output failure、successful Job not reusable、engine fatal。

Cancel requestが成立したRunは`cancelled`。

## 24. Recovery

Bootstrap -> queued pending -> activation待ち -> due Retry -> orphan running -> Lease/Review/Child -> Dynamic/reuse -> priority/concurrency repair -> Retention。

Completed RunはRecoveryだけでreopenしない。

## 25. 受入条件

1. Root system baseline/effective setting snapshot
2. Child baseline inheritance
3. root priority propagation
4. Reusable initial/retry Child concurrency reject path
5. Dynamic template no Job/Attempt
6. internal pending/claim
7. all-executor Retry pending
8. Internal ready_at / External available_at order
9. one-running/Pool
10. Maintenance due boundary
11. concurrency workflow_id+group / mixed max-runs / fairness
12. Manual Retry concurrency reacquire/output clear
13. strict current if/Input/Artifact/version validation
14. Reusable identity includes Child baseline/settings/Retention
15. Secret-bound successful Attempt reuse ineligible
16. state.get/state.set/non-input Artifact reuse ineligible
17. stored Output revalidation
18. Dynamic expansion reuse
19. recovery idempotency
