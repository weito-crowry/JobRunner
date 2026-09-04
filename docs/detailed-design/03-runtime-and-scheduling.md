# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.1
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
- **static non-dynamic Jobの`job_runs` rowだけ**作成
- Event
- optional idempotency result

Dynamic templateはDefinition snapshotに存在する仮想groupであり、Run startでは`job_runs` rowを作らない。Expansion時にgenerated concrete Jobだけを作る。Group stateは`05/08`から導出。

Static Job rowは最初`status=queued, ready_at=NULL`。Runner取得可能ではない。

Concurrency `queue|reject` は`08`。

## 4. Status / readiness

Workflow:

```text
queued|running|paused|completed
```

Conclusion:

```text
success|failure|cancelled
```

Concrete Job:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Dynamic template groupのstatus/conclusionは`05`で別途導出し、Job rowへ保存しない。

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

### `if=false`

Attempt無しでJob=`completed/skipped`。

### `if=true`

1. `continue-on-error`評価/snapshot
2. `with` -> `02` persistent Input/bindings/digest
3. executor別activation

Internal:

- pending snapshotをJob rowへ保存
- `ready_at=now`
- status queued
- Attempt無し

External:

- Attemptへsnapshot
- Task作成
- Attempt Execution Log
- waiting_external

Human:

- Attemptへsnapshot
- Review作成
- Log
- waiting_review

Reusable:

- binding確定
- Attemptへsnapshot
- Child Run作成
- Log
- waiting_child

Pre-Attempt Input/condition failureはJob-level failure。Input snapshot無しならManual Retry不可。

## 6. Dynamic template activation

RuntimeはDefinition snapshotから未確定Dynamic templateを探索する。

Activation単位:

- Root: template + Workflow Run
- Nested: template + each parent generated Job

Dependencies terminal後、`05`どおり `if -> foreach -> key/order -> atomic expansion`。

Dynamic template自身のAttempt/Execution Log/Retry targetは作らない。Expansion failureは`dynamic_expansions.failure_json` + Workflow failureで表現し、Attempt無しなので`wf_retry`直接対象にはならない。

Generated concrete Jobは通常§5のJob activationへ進む。

## 7. Selection order

Internal/External共通ordering軸:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source/generated order ASC
5. ready_at ASC
6. Job Run ID

InternalはPool exact一致。External claimも同ordering。

## 8. Atomic internal claim

Candidate:

```text
executor=internal
status=queued
ready_at non-NULL
retry_not_before NULL or <=now
pending snapshot present
same Run running internal無し
Pool一致
```

1 transactionでJob running + new Attempt snapshot copy + Runner ownership + Log/Event。

Attempt作成後pending snapshotをclear。

## 9. Queued Retry activation for non-internal executor

External/Human/Reusable Retryは`10`により:

```text
status=queued
ready_at non-NULL
pending snapshot present
retry_not_before NULL or <=now
```

となる。

Runtime schedulerはPause/Cancel gateとdueを確認し、**`with`/Inputを再評価せずpending snapshotをexact copy**してnew Attemptを作る。

- external -> new Task, waiting_external
- human -> new Review, waiting_review
- reusable -> same bindingでnew Child Run, waiting_child

Attempt作成後pending snapshot clear。

これらはRunner Poolを消費しない。

## 10. Maintenance Loop

Workflow Scheduler/Cronではなく期限/housekeeping専用。

対象:

```text
retry_not_before
external lease expires_at
concurrency wake
retention due
orphan cleanup
expired idempotency cleanup
```

Nearest deadline + condition/event wait。Busy loop禁止。

Default `maintenance_max_sleep_seconds=5`、finite positive configurable。

Retry dueはJobを新Attempt無しでwake。External LeaseはPause中もclock継続。Retentionは`01/08/09`。

Restart時overdue Retry/Lease、consistency cleanup、due Retentionを先処理。

## 11. Terminal後activation

Terminal transition後idempotentに:

- downstream concrete Job activation
- Dynamic expansion/group propagation
- Reusable parent propagation
- Result Reuse pending check
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

Direct Artifact identity=`artifact_id + optional digest`、dependency/name順canonicalization。

Executor identity:

- internal: Action ID/version
- external: `jobrunner.external_llm.v1`
- human: `jobrunner.human.v1`
- reusable: Child Definition hash + Child Action/Validator versions

Validator identity=none null or ID/version。

## 13. Reuse eligibility

原則eligible。ただし:

- Runtime Handle `state.get`
- persistent Input/direct upstream以外Artifact materialize
- executor extension explicit false

ならineligible。

Expression `state`が`with`ならinput digestへ、`if`ならcurrent再評価へ反映するのでそれだけでineligibleではない。

## 14. Manual Retry後successful Job strict reuse

保存済みInputだけ比較しない。Current dependenciesから副作用無しで再計算:

1. dependency set terminal待ち
2. current Job `if`、true必須
3. current `with` -> expected persistent Input/bindings/digest（Secret value未materialize）
4. expected digest == stored successful Attempt digest
5. current direct upstream Artifact identities一致
6. Definition/executor/Validator identity一致
7. current Registry snapshot version exact提供
8. stored Payload/required Managed Artifact integrity/availability
9. reuse_eligible=true
10. stored OutputをSchema -> Validator -> success_if -> SecretGuardで再検証

Pass -> success維持 + `job_result_reused`。

Fail -> same Runでsuccessful Jobを新Input再実行しない。`successful_job_result_not_reusable`, retryable=false, new Workflow Run要求。

Reason detail例:

```text
condition_changed
input_changed
artifact_changed
validation_changed
storage_unavailable
version_unavailable
ineligible
```

Secret materialized valueは比較しない。

## 15. Dynamic expansion reuse

Manual Retry後completed/success Dynamic groupはGenerated Job reuseより先に`05` expansion reuse check。

Exact matchのみ既存expansion保持。Mismatch/error=`dynamic_expansion_not_reusable`, new Run要求。

Whole skipped groupのみ`05/10`規則で未実行として再評価可能。

## 16. Pause / Resume

Pause:

- running internal継続
- new internal claim/noninternal Retry activation/External claim/Dynamic expansion禁止
- existing External submit/Human review/started Child進行可
- Lease expiry/Retention継続

Resume:

- unactivated `ready_at=NULL` Job/Dynamicをcurrent dependencyから再評価
- `ready_at!=NULL` queued Jobは**全executorでpending Inputを再評価しない**
- dueなら§8/§9へ

## 17. Cancel

- cancel_requested
- queued/waiting cancel
- External Lease invalidate
- running internal cancel request
- Child cancel
- new activation禁止

`always()`でもcancel後new Job開始無し。

## 18. Workflow conclusion

Success:

- required concrete Jobs/groups success
- intentionally skipped
- allowed continue-on-error failure
- reuse pending無し

Failure:

- non-allowed failure/blocked
- Dynamic expansion/group failure
- dynamic_expansion_not_reusable
- Child failure
- Workflow Output failure
- successful_job_result_not_reusable
- engine fatal

Cancel由来=cancelled。

## 19. Concurrency / Recovery

Concurrency=`08` BINARY group。

Restart:

- Bootstrap
- queued pending snapshots復元
- `ready_at=NULL` concrete Jobだけ通常activation再評価
- due noninternal Retryを§9で再開
- running internal -> runner recovery
- Lease/Review/Child
- Dynamic expansion/group
- reuse pending
- Retention

Completed RunはRecoveryだけでreopenしない。

## 20. 受入条件

1. Dynamic template has no job_runs/Attempt
2. static job start ready_at NULL
3. internal initial pending snapshot/claim copy
4. all-executor Retry queued pending snapshot
5. noninternal due Retry creates Attempt without with re-eval
6. Resume preserves all ready pending snapshots
7. one-running/ordering/Pool
8. Maintenance deadlines
9. strict successful current if/Input/Artifact/version validation
10. stored Output revalidation
11. no automatic changed-input rerun
12. Dynamic expansion reuse gate
13. state.get/non-input Artifact ineligible
14. concurrency/recovery idempotency
