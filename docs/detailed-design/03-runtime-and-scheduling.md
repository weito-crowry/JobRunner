# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.4
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
- concurrency snapshot
- static non-dynamic Job rows
- Event
- optional idempotency result

Dynamic templateはDefinition snapshotに存在する仮想group。Run startでは`job_runs` row無し。

Static Jobは最初`status=queued, ready_at=NULL`。

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

## 5. Status / readiness

Workflow=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Concrete Job=`queued|running|waiting_external|waiting_review|waiting_child|completed`、Conclusion=`success|failure|cancelled|skipped|blocked`。

Dynamic group status/conclusionは`05`で導出。

Queued concrete Job:

- `ready_at=NULL`: dependency/condition activation未完了
- `ready_at!=NULL`: 次Attempt用persistent Input snapshot確定済み
- `retry_not_before>now`: Retry backoff待ち

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
- Task + Log
- waiting_external

Human:

- Attempt snapshot
- Review + Log
- waiting_review

Reusable:

- binding
- Attempt snapshot
- Child + Log
- waiting_child

Pre-Attempt Input/condition failureはJob-level failure。Input snapshot無しならManual Retry不可。

## 7. Dynamic template activation

Definition snapshotから未確定Dynamic templateを探索。

- Root: template + Run
- Nested: template + each parent generated Job

Dependencies terminal後`05`のif/foreach/key/order/atomic expansion。

Template自身のAttempt/Execution Log/Retry target無し。Expansion failureは`dynamic_expansions.failure_json` + Workflow failure。

## 8. Selection order

Internal/External共通:

1. Workflow Run priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source/generated order ASC
5. ready_at ASC
6. Job Run ID

Child Run priorityはroot priorityと同期されるためReusable subtreeもroot運用priorityで比較される。

InternalはPool exact一致。External claimも同ordering。

## 9. Root priority update

`wf_priority_update` はroot non-terminal Runだけ許可。

1 transactionで:

- root priority更新
- `root_workflow_run_id=root.id` の全non-terminal descendant Child Run priorityを同値へ更新
- Event/idempotency result

Current running Jobをpreemptしない。Future Childは更新後root priorityを継承。

## 10. Atomic internal claim

Candidate=`internal + queued + ready_at + due + pending snapshot + same Run running internal無し + Pool一致`。

1 transactionでJob running + new Attempt snapshot copy + Runner ownership + Log/Event。Attempt後pending clear。

## 11. Queued Retry activation for non-internal executor

External/Human/Reusable Retryは:

```text
status=queued
ready_at non-NULL
pending snapshot present
retry_not_before NULL or <=now
```

SchedulerはPause/Cancel gate後、`with`を再評価せずpending snapshot exact copyでnew Attempt。

- external -> Task/waiting_external
- human -> Review/waiting_review
- reusable -> same binding Child/waiting_child

Attempt後pending clear。

## 12. Maintenance Loop

Workflow Scheduler/Cronではなく期限/housekeeping専用。

対象=`retry_not_before`, Lease expiry, concurrency wake, Retention, orphan cleanup, idempotency cleanup。

Nearest deadline + condition/event wait。Default max sleep5秒。

Pause中Lease/Retention継続。Restart時overdue/cleanup/Retention先処理。

## 13. Terminal後activation

Idempotentに:

- downstream concrete Job
- Dynamic group/expansion
- Reusable parent
- Result Reuse pending
- Workflow Output/conclusion
- concurrency release
- Workflow progress再計算

## 14. Stored Result Reuse Context

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

## 15. Reuse eligibility

Successful Attemptは原則eligible。ただし、Coreが観測できる以下のruntime依存/副作用を1回でも使ったAttemptは`reuse_eligible=false`へ落とす。

- Runtime Handle `state.get`
- Runtime Handle `state.set`
- persistent Input/direct upstream Artifact以外のArtifactを`artifact.materialize`
- executor extensionが明示`reuse_eligible=false`

`state.set` は即時永続副作用であり、成功結果だけを再利用してActionを再実行しないとstate writeを再現できないため、自動reuse対象にしない。

Expressionの`state`利用は、それが`with`ならInput digestへ入り、`if`なら§16でcurrent再評価されるため、それだけでineligibleにはしない。

## 16. Manual Retry後successful Job strict reuse

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

## 17. Dynamic expansion reuse

Manual Retry後completed/success Dynamic groupはGenerated Job reuseより前に`05` expansion digest check。Exact matchのみ保持。Mismatch=`dynamic_expansion_not_reusable`。

Whole skipped groupのみ未実行として再評価可能。

## 18. Pause / Resume

Pause:

- running internal継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing submit/review/started Child進行可
- Lease/Retention継続

Resume:

- ready_at NULL concrete Job/Dynamicをcurrent dependencyから再評価
- ready_at non-NULL queued Jobは全executorでpending Input再評価無し
- dueなら§10/§11

## 19. Cancel

queued/waiting cancel、Lease invalidate、running internal cancel、Child cancel、new activation禁止。Cancel後alwaysでもnew Job無し。

## 20. Workflow conclusion

Success=required concrete Jobs/groups success、intentional skip、allowed continue-on-error、reuse pending無し。

Failure=non-allowed failure/blocked、Dynamic failure/not-reusable、Child failure、Workflow Output failure、successful Job not reusable、engine fatal。

Cancel由来=cancelled。

## 21. Concurrency / Recovery

Concurrency=`08` BINARY group。

Restart:

- Bootstrap
- queued pending snapshots
- ready_at NULL concrete Jobだけ通常activation再評価
- due noninternal Retry
- running internal recovery
- Lease/Review/Child
- Dynamic/reuse pending
- root/Child priority整合repair
- Retention

Completed RunはRecoveryだけでreopenしない。

## 22. 受入条件

1. Root system baseline/effective setting snapshot
2. Child inherits Parent Run baseline, not current System
3. root priority resolution/update/descendant propagation
4. Dynamic template no Job/Attempt
5. static ready_at NULL
6. internal pending/claim copy
7. all-executor Retry pending
8. noninternal Retry no with re-eval
9. one-running/ordering/Pool
10. Maintenance
11. strict current if/Input/Artifact/version validation
12. Reusable identity includes Child baseline/settings/Retention
13. stored Output revalidation
14. no changed-input rerun
15. Dynamic expansion reuse
16. state.get marks reuse ineligible
17. state.set marks reuse ineligible
18. non-input Artifact materialize marks reuse ineligible
19. concurrency/recovery idempotency
