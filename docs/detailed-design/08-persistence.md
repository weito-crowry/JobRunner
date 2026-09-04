# 08. Persistence 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03-runtime-and-scheduling.md`〜`07-external-and-human.md`, `10-retry-recovery-cancel.md`

## 1. 目的

SQLite schema、ID、transaction、unique/index、Migration、Recovery、Idempotencyの正規契約を定義する。

## 2. 基本原則

1. MVP標準backendは専用SQLite DB。
2. persistence interfaceは将来PostgreSQLへ交換可能。
3. 状態遷移・関連Event・idempotency resultは可能な限り同一transaction。
4. Execution Log本文/Artifact実体はDBへ大容量保存しない。
5. append-only履歴を上書きしない。
6. SQLite writer競合を前提に短いtransaction + conditional updateを使う。

## 3. SQLite設定

Connectionごとに:

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=<system configurable; default 5000ms>
```

DB初期化時:

```text
PRAGMA journal_mode=WAL
```

Scheduling/claim/concurrency等、競合するwrite判断は必要に応じて`BEGIN IMMEDIATE`でwriter順序を確定する。長時間処理中にtransactionを保持しない。

## 4. ID

外部公開IDは **type prefix + UUID4 hex** をcanonicalとする。Python標準libraryのみで生成可能。

例:

```text
wr_550e8400e29b41d4a716446655440000
jr_...
att_...
exp_...
art_...
task_...
lease_...
review_...
bind_...
evt_...
```

IDの辞書順に時系列性を期待しない。Orderingは`created_at`等の明示columnを使う。

## 5. 時刻 / JSON

- UTC timezone-aware datetimeをrepository境界で使用。
- SQLiteはcanonical RFC3339 UTC stringを基本とする。
- JSONはUTF-8、object key sort可能、NaN/Infinity禁止。
- Hash対象はcanonical serialization。

## 6. MVP tables

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

`artifact_aliases` / `workflow_concurrency`専用tableはMVPでは作らない。current ArtifactはAttempt/created_atから解決し、Concurrencyは`workflow_runs` snapshot fieldsから集計する。

## 7. `workflow_runs`

主要column:

```text
id TEXT PK
workflow_id TEXT NOT NULL
workflow_version INTEGER NOT NULL CHECK >=1
status TEXT NOT NULL
conclusion TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
run_attempt INTEGER NOT NULL DEFAULT 1 CHECK >=1
wait_reason TEXT NULL
pause_requested INTEGER NOT NULL DEFAULT 0 CHECK IN(0,1)
cancel_requested INTEGER NOT NULL DEFAULT 0 CHECK IN(0,1)

definition_yaml TEXT NOT NULL
definition_json TEXT NOT NULL
definition_hash TEXT NOT NULL
input_json TEXT NOT NULL
action_versions_json TEXT NOT NULL
output_json TEXT NULL
failure_json TEXT NULL
source_identity TEXT NULL

concurrency_group TEXT NULL
concurrency_max_runs INTEGER NULL
concurrency_on_limit TEXT NULL

actor_context_json TEXT NULL
access_scope_json TEXT NULL

parent_workflow_run_id TEXT NULL FK
parent_job_run_id TEXT NULL FK
parent_attempt_id TEXT NULL FK
root_workflow_run_id TEXT NOT NULL FK
call_depth INTEGER NOT NULL DEFAULT 0
reusable_binding_id TEXT NULL FK

created_at TEXT NOT NULL
started_at TEXT NULL
completed_at TEXT NULL
updated_at TEXT NOT NULL
```

Checks:

- status `queued|running|paused|completed`
- conclusion null unless completed; when present `success|failure|cancelled`
- concurrency fields all null or valid set; `max_runs>=1`, on_limit `queue|reject`

Indexes:

```text
(status, priority DESC, created_at)
(workflow_id, created_at DESC)
(concurrency_group, status, wait_reason)
(parent_workflow_run_id)
(root_workflow_run_id)
```

## 8. `job_runs`

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
job_key TEXT NOT NULL
job_template_key TEXT NULL
dynamic_key TEXT NULL
parent_job_run_id TEXT NULL FK
dynamic_expansion_id TEXT NULL FK
item_snapshot_json TEXT NULL
iteration_context_json TEXT NULL
source_order INTEGER NULL
order_rank INTEGER NULL

executor TEXT NOT NULL
action_id TEXT NULL
action_version TEXT NULL
uses_ref TEXT NULL
reusable_binding_id TEXT NULL FK
runs_on TEXT NULL

status TEXT NOT NULL
conclusion TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
continue_on_error INTEGER NULL CHECK IN(0,1)
timeout_seconds REAL NULL
retry_policy_json TEXT NULL
retry_not_before TEXT NULL
current_attempt_no INTEGER NOT NULL DEFAULT 0

input_json TEXT NULL
output_json TEXT NULL
failure_json TEXT NULL
ready_at TEXT NULL
queued_at TEXT NULL
started_at TEXT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

Unique:

```text
UNIQUE(workflow_run_id, job_key)
```

status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

conclusion:

```text
NULL|success|failure|cancelled|skipped|blocked
```

### 8.1 1 Workflow Run internal running最大1

SQLite partial unique index:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status = 'running' AND executor = 'internal';
```

Application checkに加えDBでも保証する。

## 9. `job_attempts`

```text
id TEXT PK
job_run_id TEXT NOT NULL FK
attempt_no INTEGER NOT NULL CHECK >=1
status TEXT NOT NULL
conclusion TEXT NULL
runner_id TEXT NULL
runner_instance_id TEXT NULL
runtime_instance_id TEXT NULL
input_json TEXT NOT NULL
output_json TEXT NULL
failure_json TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
UNIQUE(job_run_id, attempt_no)
```

Attemptは実際のexecution開始時だけ作る。Retry backoff予約時には作らない。

statusは`running|waiting_external|waiting_review|waiting_child|completed`。conclusionはJobと同じterminal subset。

## 10. `job_steps`

```text
id TEXT PK
attempt_id TEXT NOT NULL FK
sequence INTEGER NOT NULL
name TEXT NOT NULL
status TEXT NOT NULL
conclusion TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
metadata_json TEXT NULL
UNIQUE(attempt_id, sequence)
```

## 11. `dynamic_expansions`

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
template_id TEXT NOT NULL
parent_job_run_id TEXT NULL FK
source_snapshot_json TEXT NOT NULL
source_digest TEXT NOT NULL
generated_count INTEGER NOT NULL DEFAULT 0
status TEXT NOT NULL
failure_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

Root/Nestedのexact uniquenessにはnullable parentの扱いを明示するため2 partial unique indexを使う。

```sql
CREATE UNIQUE INDEX uq_dynamic_root_expansion
ON dynamic_expansions(workflow_run_id, template_id)
WHERE parent_job_run_id IS NULL;

CREATE UNIQUE INDEX uq_dynamic_nested_expansion
ON dynamic_expansions(workflow_run_id, template_id, parent_job_run_id)
WHERE parent_job_run_id IS NOT NULL;
```

status: `building|completed|failed|skipped`。

## 12. `reusable_bindings`

```text
id TEXT PK
parent_workflow_run_id TEXT NOT NULL FK
parent_job_run_id TEXT NOT NULL FK UNIQUE
workflow_ref_original TEXT NOT NULL
child_workflow_id TEXT NOT NULL
child_workflow_version INTEGER NOT NULL
child_definition_yaml TEXT NOT NULL
child_definition_json TEXT NOT NULL
child_definition_hash TEXT NOT NULL
child_action_versions_json TEXT NOT NULL
created_at TEXT NOT NULL
```

Child Workflow Run側の`parent_attempt_id`にunique partial indexを置き、同じParent AttemptへChildを1件にする。

```sql
CREATE UNIQUE INDEX uq_child_run_per_parent_attempt
ON workflow_runs(parent_attempt_id)
WHERE parent_attempt_id IS NOT NULL;
```

## 13. Workflow State

`workflow_state`:

```text
workflow_run_id TEXT FK
name TEXT
revision INTEGER >=1
value_json TEXT
updated_at TEXT
updated_by_job_run_id TEXT NULL
updated_by_attempt_id TEXT NULL
updated_by_step_id TEXT NULL
PRIMARY KEY(workflow_run_id, name)
```

`workflow_state_history` append-only:

```text
id TEXT PK
workflow_run_id TEXT
name TEXT
revision INTEGER
old_value_json TEXT NULL
new_value_json TEXT NULL
job_run_id TEXT NULL
attempt_id TEXT NULL
step_id TEXT NULL
created_at TEXT
UNIQUE(workflow_run_id, name, revision)
```

Current update + history insertを同一transaction。

## 14. Artifacts

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK
name TEXT NOT NULL
uri TEXT NOT NULL
media_type TEXT NULL
size_bytes INTEGER NULL CHECK size_bytes>=0
digest TEXT NULL
metadata_json TEXT NULL
created_at TEXT NOT NULL
deleted_at TEXT NULL
```

Artifactはimmutable。Currentはcurrent successful Attempt内で`name`一致 + created_at/id順の最新非deleted rowから解決する。Alias tableは使わない。

## 15. Events

```text
id TEXT PK
workflow_run_id TEXT NULL
job_run_id TEXT NULL
attempt_id TEXT NULL
runner_id TEXT NULL
event_type TEXT NOT NULL
actor_type TEXT NULL
actor_id TEXT NULL
source TEXT NULL
payload_json TEXT NOT NULL
created_at TEXT NOT NULL
```

Indexes `(workflow_run_id, created_at,id)`, `(job_run_id,created_at,id)`, `(event_type,created_at,id)`。

## 16. Execution Logs

```text
id TEXT PK
attempt_id TEXT NOT NULL FK UNIQUE
relative_path TEXT NOT NULL
size_bytes INTEGER NOT NULL DEFAULT 0
first_written_at TEXT NULL
last_written_at TEXT NULL
created_at TEXT NOT NULL
```

`relative_path`はdata root配下の内部ID由来path。外部入力pathを保存しない。

## 17. Runners / restart

`runners`:

```text
runner_id TEXT PK
runner_instance_id TEXT NOT NULL
runtime_instance_id TEXT NOT NULL
pool_name TEXT NOT NULL
status TEXT NOT NULL
pid INTEGER NULL
current_job_run_id TEXT NULL
current_attempt_id TEXT NULL
started_at TEXT NOT NULL
last_heartbeat_at TEXT NOT NULL
main_loop_tick_at TEXT NOT NULL
stopped_at TEXT NULL
metadata_json TEXT NULL
```

`runner_restarts`:

```text
id TEXT PK
runner_id TEXT NOT NULL
old_instance_id TEXT NULL
new_instance_id TEXT NULL
reason TEXT NOT NULL
created_at TEXT NOT NULL
```

Restart回数/windowはtimestampsから計算する。

## 18. External Tasks / Leases

`external_tasks`:

```text
id TEXT PK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK UNIQUE
status TEXT NOT NULL
input_json TEXT NOT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

`external_leases`:

```text
id TEXT PK
external_task_id TEXT NOT NULL FK
lease_id TEXT NOT NULL UNIQUE
claimed_by TEXT NOT NULL
status TEXT NOT NULL
claimed_at TEXT NOT NULL
expires_at TEXT NOT NULL
released_at TEXT NULL
```

1 Task active lease最大1:

```sql
CREATE UNIQUE INDEX uq_external_active_lease
ON external_leases(external_task_id)
WHERE status = 'active';
```

## 19. Human Reviews

```text
id TEXT PK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK UNIQUE
status TEXT NOT NULL
outcome TEXT NULL
comment TEXT NULL
actor_context_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

Check:

- status `pending|completed|cancelled`
- pending -> outcome null
- completed -> outcome `approve|reject`
- cancelled -> outcome null

## 20. Idempotency

```text
scope TEXT NOT NULL
operation TEXT NOT NULL
request_key TEXT NOT NULL
request_hash TEXT NOT NULL
result_json TEXT NOT NULL
status TEXT NOT NULL
created_at TEXT NOT NULL
expires_at TEXT NOT NULL
PRIMARY KEY(scope, operation, request_key)
```

System default TTL = 24 hours。設定変更可能。

- TTL内 same key/hash -> stored result replay
- TTL内 same key/different hash -> conflict
- expiry後はrecordをretention可能で、古いkeyへのreplay protectionを保証しない
- request hashへSecret valueを含めない

Side effectとidempotency resultは同一transactionに置く。External `claim_next`の「next claim」はsubmit本体とは別transaction。

## 21. Concurrency transaction

Workflow start (`on-limit`) とslot releaseは`BEGIN IMMEDIATE` transaction内で同一`concurrency_group`のholder数を再集計し、max-runsを超えてactive holderを作らない。

Workflow Run rowにgroup/max/on_limit snapshotを持つため専用table不要。

## 22. Manual Retry reopen transaction

`10`のexplicit manual retryでcompleted failed Runを再openする場合1transactionで:

- authorization/idempotency確認
- `run_attempt += 1`
- Workflow Run status -> queued/running、conclusion/failure/completed_at clear
- target failed Job retry stateへ
- failure由来blocked descendantsを再評価可能状態へ
- Event + idempotency result

Attempt/Event履歴は削除しない。

## 23. Transaction境界

最低限atomic:

- Workflow start + Jobs + Event + idempotency
- internal claim + Attempt + Runner ownership
- Dynamic expansion all rows
- Reusable binding / Child relation
- External activation / claim / submit
- Human activation / submit
- state current + history
- auto retry scheduling
- manual retry reopen
- cancel

## 24. Conditional update / fencing

例:

```sql
UPDATE job_runs
SET status='running'
WHERE id=? AND status='queued';
```

0 rowsならconflict/re-read。

Runner completionは`runtime_instance_id + runner_instance_id + attempt_id + current status`一致を必要とする。

## 25. Migration

`migrations/NNN_name.sql` + `schema_migrations(version, applied_at)`。

順序通り1回。失敗はfail-closed。可能な限り1 migration transaction。SQLiteでtransaction外DDLが必要な場合も途中状態を検出可能にする。

## 26. Retention / Backup

Retention既定無期限。削除はFK順序を守りEventを残す。Artifact実体は親責任。

BackupはMVP専用manager無し。SQLite file + data rootを親側がbackupできる構造。DB保存filesystem pathはdata rootからのrelative pathを基本とする。

## 27. Repository boundary

```text
WorkflowRunRepository
JobRepository
DynamicExpansionRepository
ReusableBindingRepository
RunnerRepository
ExternalTaskRepository
HumanReviewRepository
ArtifactRepository
StateRepository
EventRepository
IdempotencyRepository
```

Runtime logicからbackend固有SQLを隔離する。

## 28. 受入条件

1. fresh/incremental migration
2. UUID4 prefix ID
3. FK/check integrity
4. internal running partial unique
5. dynamic root/nested uniqueness/rollback
6. reusable binding + child per attempt uniqueness
7. External Task/active lease uniqueness
8. Human pending/outcome constraint
9. state current/history atomic
10. concurrency start/release race
11. idempotency same transaction + 24h expiry
12. manual retry reopen atomic
13. fencing conditional update
14. safe relative log path
15. WAL concurrent read/write
