# 08. Persistence 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

SQLite schema、transaction、unique/index、Migration、Recovery、Idempotencyの正規契約を定義する。

## 2. 基本原則

1. MVP標準backendは専用SQLite DB。
2. persistence interfaceは将来PostgreSQLへ交換可能。
3. 状態遷移・Event・idempotency resultは可能な限り同一transaction。
4. Execution Log本文 / Artifact実体は大容量DB保存しない。
5. append-only履歴は上書きしない。
6. SQLite writer競合は短transaction + conditional updateで扱う。

## 3. SQLite設定

Connectionごと:

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000  # default, configurable
```

DB初期化時:

```text
PRAGMA journal_mode=WAL
```

競合するwrite判断では必要に応じ `BEGIN IMMEDIATE`。

## 4. ID / 時刻

外部公開IDは type prefix + UUID4 hex。

時刻はUTC RFC3339。JSONはUTF-8、NaN/Infinity禁止。

## 5. MVP tables

```text
schema_migrations
workflow_runs
job_runs
job_attempts
job_steps
dynamic_expansions
reusable_bindings
workflow_state
workflow_state_history
artifacts
events
execution_logs
runners
runner_restarts
external_tasks
external_leases
human_reviews
idempotency_records
```

## 6. `workflow_runs`

主要column:

```text
id
workflow_id/workflow_version
status/conclusion
priority
run_attempt
wait_reason
pause_requested/cancel_requested

definition_yaml/definition_json/definition_hash
input_json/action_versions_json/output_json/failure_json
source_identity

concurrency_group/concurrency_max_runs/concurrency_on_limit

actor_context_json/access_scope_json
parent_workflow_run_id/parent_job_run_id/parent_attempt_id
root_workflow_run_id/call_depth/reusable_binding_id

created_at/started_at/completed_at/updated_at
```

status: `queued|running|paused|completed`。
conclusion: `success|failure|cancelled` when completed。

## 7. `job_runs`

```text
id
workflow_run_id
job_key
job_template_key
dynamic_key
parent_job_run_id
dynamic_expansion_id
item_snapshot_json/iteration_context_json
source_order/order_rank

executor/action_id/action_version/uses_ref/reusable_binding_id/runs_on
status/conclusion
priority/continue_on_error
timeout_seconds
retry_policy_json/retry_not_before/current_attempt_no
input_json/output_json/failure_json
ready_at/queued_at/started_at/completed_at/created_at/updated_at
```

`UNIQUE(workflow_run_id, job_key)`。

status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

### 7.1 internal running最大1

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
```

## 8. `job_attempts`

```text
id
job_run_id
attempt_no
status/conclusion
runner_id/runner_instance_id/runtime_instance_id
input_json/output_json/failure_json
started_at/completed_at/created_at
```

`UNIQUE(job_run_id, attempt_no)`。

Attemptは実行開始時だけ作る。Retry backoff予約時には作らない。

## 9. Dynamic expansion

`dynamic_expansions`:

```text
id
workflow_run_id
template_id
parent_job_run_id nullable
source_snapshot_json/source_digest
generated_count
status/failure_json
created_at/completed_at
```

Root/Nested uniqueness:

```sql
CREATE UNIQUE INDEX uq_dynamic_root_expansion
ON dynamic_expansions(workflow_run_id, template_id)
WHERE parent_job_run_id IS NULL;

CREATE UNIQUE INDEX uq_dynamic_nested_expansion
ON dynamic_expansions(workflow_run_id, template_id, parent_job_run_id)
WHERE parent_job_run_id IS NOT NULL;
```

full logical `job_key` は SQLite `TEXT`。固定byte長CHECKを設けない。

## 10. Reusable binding

one binding / Parent Job Run。

Child Workflow Runは `parent_attempt_id IS NOT NULL` にunique partial indexを置き one Child / Parent Attempt。

## 11. State / Artifact / Event / Log

Workflow stateはcurrent table + append-only historyを同transactionで更新。

Artifactはimmutable metadata row。current Artifactはcurrent successful Attempt内の最新 non-deleted rowを解決する。

Eventはappend-only。

Execution Log DBはrelative path/size/time metadataのみ。

## 12. Runner fencing

`runners` は `runner_id / runner_instance_id / runtime_instance_id / pool / status / current job/attempt / heartbeat / main_loop_tick` を持つ。

Runner completionは current `runtime_instance_id + runner_instance_id + attempt_id` 一致を条件にする。

## 13. External / Human uniqueness

- `external_tasks.attempt_id UNIQUE`
- active LeaseはTaskごと最大1のpartial unique index
- `human_reviews.attempt_id UNIQUE`
- pending review outcomeはnull、completedはapprove/reject

## 14. Idempotency record

```text
scope TEXT
operation TEXT
request_key TEXT
request_hash TEXT
result_json TEXT
status TEXT
created_at TEXT
expires_at TEXT
PRIMARY KEY(scope, operation, request_key)
```

Default TTL = 24 hours、設定可能。

### 14.1 scope の正規構成

Service layerが以下からcanonical scope stringを作る。

```text
system_namespace
operation resource scope
AccessScope canonical JSON/hash
Actor isolation key
```

Actor isolation key:

- authenticated actor_id がある: `actor:<actor_type>:<actor_id>`
- system/runner内部操作: Runtimeが発行する安定した内部principal
- actor_id無しの外部clientをstate-changing public operationへ通す場合、親Adapterがclient/session principalを補う

**異なるActor/AccessScope間で同じrequest_idを共有しない。**

### 14.2 request hash

Secret値は含めず、Secret reference markerを使う。canonical request modelをhashする。

### 14.3 TTL内

- same key + same hash -> stored result replay
- same key + different hash -> `idempotency_conflict`

### 14.4 TTL期限後のkey再利用

期限切れrecordがDBに残っていても、新request処理transaction内で:

1. expired recordを検出
2. 旧recordを削除またはhistory retention対象へ退避
3. **同じtransaction内で新record用slotを取得**
4. 新side effect + 新resultを確定

とする。

単にPK conflictで新requestを拒否してはならない。

TTL後は過去side effectへのreplay protectionを保証しないため、同じrequest_idは「新しいrequest id」として扱われ得る。

## 15. Concurrency transaction

Workflow start とslot releaseは `BEGIN IMMEDIATE` 内で同group holder数を再集計し max-runs超過を防ぐ。

## 16. Manual Retry reopen

Completed failed Runをmanual retryする場合:

- `run_attempt += 1`
- Run statusを再open
- target Jobをretry待機へ
- failure由来blocked descendantsを再評価可能状態へ
- Attempt/Event履歴保持

同transactionにAuthorization/idempotency/Eventを含める。

## 17. Atomic境界

最低限:

- Workflow start + initial Jobs
- internal claim + Attempt + Runner ownership
- Dynamic expansion all rows
- Reusable binding/Child relation
- External activation/claim/submit
- Human activation/submit
- state current+history
- auto retry scheduling
- manual retry reopen
- cancel
- idempotency expired replacement

## 18. Migration

`migrations/NNN_name.sql` + `schema_migrations`。順序通り1回、失敗時fail-closed。

## 19. Retention

既定無期限。削除はFK順序を守りEventを残す。Artifact実体は親責任。

Idempotency expired recordはmaintenanceで削除可能だが、前節のtransactional replacementによりmaintenance未実行でもkey再利用可能。

## 20. 受入条件

1. WAL/FK/busy timeout
2. internal running unique
3. Dynamic root/nested unique + no job_key length CHECK
4. Reusable uniqueness
5. External/Human uniqueness
6. concurrency race
7. idempotency actor/scope isolation
8. TTL内 replay/conflict
9. TTL後、expired rowが残った状態でsame key再利用
10. manual retry reopen
11. old Runner fencing
