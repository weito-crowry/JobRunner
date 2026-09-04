# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `05`, `08`, `09`, `10`

## 1. 基本原則

1. Runnerが取得する単位はconcrete internal Job。
2. RunnerはWorkflow Runを占有しない。
3. 同一Workflow Run internal Job同時実行最大1。
4. 別Workflow RunはRunner Pool空きまで並列。
5. External/Human/Reusable待ちはRunner非占有。
6. Dynamic templateは仮想groupでありRunner取得対象/Attempt対象ではない。
7. DB永続状態がSource of Truth。
8. claim/state transitionはconditional transaction。
9. Priority変更はpreemptしない。
10. Job result自動再利用は同一Workflow Run内だけ。別Run/global cache無し。
11. External Lease/Retry backoff/Retention等の期限処理はRuntime Maintenance Loop。

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

## 3. Workflow Run start

Start前にDefinition/Input/Registry version/Runner Pool/Expression/Reusable/Concurrency/Authorizationを検証。

Start transaction:

- Run Definition/Input/effective settings/Retention snapshot
- Action/Validator version snapshot
- concurrency snapshot
- static non-dynamic Jobの`job_runs` rowだけ作成
- Event
- optional idempotency result

Dynamic templateはDefinition snapshotに存在する仮想group。Run startで`job_runs` rowを作らず、Expansion時にgenerated concrete Jobだけを作る。Group stateは`05/08`から導出。

Static Job rowは最初`status=queued, ready_at=NULL`。Runner取得可能ではない。

Concurrency `queue|reject` は`08`。

## 4. Status / readiness

Workflow=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Concrete Job=`queued|running|waiting_external|waiting_review|waiting_child|completed`、Conclusion=`success|failure|cancelled|skipped|blocked`。

Dynamic template groupのstatus/conclusionは`05`で導出しJob rowへ保存しない。

Queued concrete Job:

- `ready_at=NULL`: dependency/condition activation未完了
- `ready_at!=NULL`: 次Attempt用persistent Input snapshot確定済み
- `retry_not_before>now`: Retry backoff待ち

## 5. Concrete Job初回activation

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

## 6. Dynamic template activation

RuntimeはDefinition snapshotから未確定Dynamic templateを探索。

- Root: template + Run
- Nested: template + each parent generated Job

Dependencies terminal後`05`のif/foreach/key/order/atomic expansion。

Template自身のAttempt/Execution Log/Retry target無し。Expansion failureは`dynamic_expansions.failure_json` + Workflow failure。Generated concrete Jobは通常§5へ。

## 7. Selection order

Internal/External共通:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source/generated order ASC
5. ready_at ASC
6. Job Run ID

InternalはPool exact一致。External claimも同ordering。

## 8. Atomic internal claim

Candidate=`internal + queued + ready_at + due + pending snapshot + same Run running internal無し + Pool一致`。

1 transactionでJob running + new Attempt snapshot copy + Runner ownership + Log/Event。Attempt後pending clear。

## 9. Queued Retry activation for non-internal executor

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

Attempt後pending clear。Runner非消費。

## 10. Maintenance Loop

Workflow Scheduler/Cronではなく期限/housekeeping専用。

対象=`retry_not_before`, Lease expiry, concurrency wake, Retention, orphan cleanup, idempotency cleanup。

Nearest deadline + condition/event wait。Default max sleep5秒。

Pause中Lease/Retention継続。Restart時overdue/cleanup/Retention先処理。

## 11. Terminal後activation

Idempotentに:

- downstream concrete Job
- Dynamic group/expansion
- Reusable parent
- Result Reuse pending
- Workflow Output/conclusion
- concurrency release
- Workflow progress再計算

## 12. Stored Result Reuse Context

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
  - Child effective settings JSON
  - Child retention policy JSON

Validator identity=none null or ID/version。

Reusable bindingのSystem-derived settingsもidentityへ含めるため、同じChild Definitionでも実行条件が違うbindingを同一結果扱いしない。

## 13. Reuse eligibility

原則eligible。ただしRuntime `state.get`、persistent Input/direct upstream以外Artifact materialize、executor extension falseでineligible。

Expression stateがwithならInput digest、ifならcurrent再評価へ反映するのでそれだけでineligibleではない。

## 14. Manual Retry後successful Job strict reuse

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
10. stored Output Schema -> Validator -> success_if -> SecretGuard再検証

Pass -> success維持 + `job_result_reused`。

Fail -> `successful_job_result_not_reusable`, retryable=false, new Workflow Run要求。Same Runでsuccessful Jobをnew Input自動再実行しない。

## 15. Dynamic expansion reuse

Manual Retry後completed/success Dynamic groupはGenerated Job reuseより前に`05` expansion digest check。Exact matchのみ保持。Mismatch=`dynamic_expansion_not_reusable`。

Whole skipped groupのみ未実行として再評価可能。

## 16. Pause / Resume

Pause:

- running internal継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing submit/review/started Child進行可
- Lease/Retention継続

Resume:

- ready_at NULL concrete Job/Dynamicをcurrent dependencyから再評価
- ready_at non-NULL queued Jobは全executorでpending Input再評価無し
- dueなら§8/§9

## 17. Cancel

queued/waiting cancel、Lease invalidate、running internal cancel、Child cancel、new activation禁止。Cancel後alwaysでもnew Job無し。

## 18. Workflow conclusion

Success=required concrete Jobs/groups success、intentional skip、allowed continue-on-error、reuse pending無し。

Failure=non-allowed failure/blocked、Dynamic failure/not-reusable、Child failure、Workflow Output failure、successful Job not reusable、engine fatal。

Cancel由来=cancelled。

## 19. Concurrency / Recovery

Concurrency=`08` BINARY group。

Restart:

- Bootstrap
- queued pending snapshots
- ready_at NULL concrete Jobだけ通常activation再評価
- due noninternal Retry
- running internal recovery
- Lease/Review/Child
- Dynamic/reuse pending
- Retention

Completed RunはRecoveryだけでreopenしない。

## 20. 受入条件

1. Dynamic template no Job/Attempt
2. static ready_at NULL
3. internal pending/claim copy
4. all-executor Retry pending
5. noninternal Retry no with re-eval
6. Resume pending invariant
7. one-running/ordering/Pool
8. Maintenance
9. strict current if/Input/Artifact/version validation
10. Reusable identity includes Child settings/Retention
11. stored Output revalidation
12. no changed-input rerun
13. Dynamic expansion reuse
14. state.get/non-input Artifact ineligible
15. concurrency/recovery idempotency
