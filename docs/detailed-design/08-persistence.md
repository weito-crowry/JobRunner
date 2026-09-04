# 08. Persistence 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- Canonical JSON: `01-workflow-definition.md` の `jobrunner.canonical-json.v1`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Action/Validator snapshot、Dynamic expansion、Result Reuse、deadline、Retention、Migration、Recovery、Idempotencyを実装可能な正規契約として定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ Payload metadata
├─ Artifact metadata
├─ Action/Validator identity
└─ Dynamic/Reuse/idempotency/retention metadata

PayloadStore
└─ large JSON Output blob

ArtifactStore
└─ explicitly managed artifact data
```

MVP standard=`sqlite3`, `LocalHybridPayloadStore`, `LocalArtifactStore`。

## 3. SQLite共通規則

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000
PRAGMA journal_mode=WAL
```

- 競合writeは必要時`BEGIN IMMEDIATE`
- 長transaction禁止
- FK原則`ON DELETE NO ACTION`
- implicit cascade無し
- Text identityはSQLite BINARY semantics
- TimestampはUTC RFC3339 TEXT
- IDはtype prefix + UUID4 hex TEXT
- 外部入力INTEGERはsigned 64-bit範囲
- JSON永続化/digestはcanonical-json-v1
- booleanはINTEGER `0|1`

Retention deleteはFKを無効化せず明示順序で行う。

## 4. Canonical table set

MVP migrationが作成するtableは**18個**。

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

Workflow Definition履歴tableは作らない。Definition sourceはResolver、実行済みDefinitionは`workflow_runs` snapshotがSource of Truth。

## 5. Payload column set

Workflow/Attempt Output共通:

```text
output_storage_kind TEXT NULL       # inline|blob
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
```

Invariant:

```text
none: all NULL
inline: kind=inline, json non-NULL, blob_key NULL, size>=0, digest non-empty
blob: kind=blob, json NULL, blob_key non-empty, size>=0, digest non-empty
```

## 6. `schema_migrations`

```text
version INTEGER PRIMARY KEY CHECK(version>=1)
name TEXT NOT NULL CHECK(length(name)>0)
applied_at TEXT NOT NULL
```

## 7. `workflow_runs`

```text
id TEXT PRIMARY KEY
workflow_id TEXT NOT NULL CHECK(length(workflow_id)>0)
workflow_version INTEGER NOT NULL CHECK(workflow_version>=1)
name TEXT NOT NULL CHECK(length(name)>0)
status TEXT NOT NULL
conclusion TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
run_attempt INTEGER NOT NULL DEFAULT 1 CHECK(run_attempt>=1)
wait_reason TEXT NULL
pause_requested INTEGER NOT NULL DEFAULT 0 CHECK(pause_requested IN(0,1))
cancel_requested INTEGER NOT NULL DEFAULT 0 CHECK(cancel_requested IN(0,1))
definition_yaml TEXT NOT NULL
definition_json TEXT NOT NULL
definition_hash TEXT NOT NULL CHECK(length(definition_hash)>0)
input_json TEXT NOT NULL
action_versions_json TEXT NOT NULL
validator_versions_json TEXT NOT NULL
retention_policy_json TEXT NOT NULL
output_storage_kind TEXT NULL
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
failure_json TEXT NULL
source_identity TEXT NULL
concurrency_group TEXT COLLATE BINARY NULL
concurrency_max_runs INTEGER NULL
concurrency_on_limit TEXT NULL
actor_context_json TEXT NOT NULL
access_scope_json TEXT NOT NULL
parent_workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
parent_job_run_id TEXT NULL
parent_attempt_id TEXT NULL
root_workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
call_depth INTEGER NOT NULL DEFAULT 0 CHECK(call_depth>=0)
reusable_binding_id TEXT NULL
created_at TEXT NOT NULL
started_at TEXT NULL
completed_at TEXT NULL
updated_at TEXT NOT NULL
```

`parent_job_run_id/parent_attempt_id/reusable_binding_id` は循環FKを避ける**論理pointer**。Child作成transactionでRepositoryがparent Job/Attempt/Bindingの存在・対応関係を検証する。

Enum/check:

```text
status=queued|running|paused|completed
conclusion=NULL|success|failure|cancelled
completed <=> conclusion non-NULL
source_identity NULL or length>0
concurrency_group NULL or length>0
concurrency_max_runs NULL or >=1
concurrency_on_limit NULL|queue|reject
concurrency_group NULL <=> max/on_limit NULL
root Run: parent_workflow_run_id NULL, call_depth=0, root_workflow_run_id=id
Child: parent_workflow_run_id non-NULL, parent_job/attempt/binding non-empty, call_depth>=1
```

Retention policy keys:

```text
run_history_days: integer>=1|null
execution_logs_days: integer>=1|null
event_days: integer>=1|null
artifact_metadata_days: integer>=1|null
managed_artifact_data_days: integer>=1|null
```

Indexes:

```sql
CREATE INDEX ix_workflow_runs_list
ON workflow_runs(created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_by_workflow
ON workflow_runs(workflow_id,created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_status
ON workflow_runs(status,priority DESC,created_at ASC,id ASC);
CREATE INDEX ix_workflow_runs_concurrency
ON workflow_runs(concurrency_group,status)
WHERE concurrency_group IS NOT NULL AND status!='completed';
CREATE UNIQUE INDEX uq_child_run_per_parent_attempt
ON workflow_runs(parent_attempt_id)
WHERE parent_attempt_id IS NOT NULL;
```

## 8. `job_runs`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_key TEXT NOT NULL CHECK(length(job_key)>0)
job_template_key TEXT NOT NULL CHECK(length(job_template_key)>0)
dynamic_key TEXT NULL
parent_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
dynamic_expansion_id TEXT NULL
item_snapshot_json TEXT NULL
iteration_context_json TEXT NULL
source_order INTEGER NOT NULL DEFAULT 0 CHECK(source_order>=0)
order_rank INTEGER NULL CHECK(order_rank>=0)
executor TEXT NOT NULL
action_id TEXT NULL
action_version TEXT NULL
validator_id TEXT NULL
validator_version TEXT NULL
uses_ref TEXT NULL
reusable_binding_id TEXT NULL
runs_on TEXT NULL
status TEXT NOT NULL
conclusion TEXT NULL
terminal_reason TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
continue_on_error INTEGER NOT NULL DEFAULT 0 CHECK(continue_on_error IN(0,1))
timeout_seconds REAL NULL
retry_policy_json TEXT NOT NULL
retry_not_before TEXT NULL
current_attempt_no INTEGER NOT NULL DEFAULT 0 CHECK(current_attempt_no>=0)
current_attempt_id TEXT NULL
reuse_check_pending INTEGER NOT NULL DEFAULT 0 CHECK(reuse_check_pending IN(0,1))
current_failure_json TEXT NULL
ready_at TEXT NULL
queued_at TEXT NULL
started_at TEXT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(workflow_run_id,job_key)
```

`dynamic_expansion_id/reusable_binding_id/current_attempt_id` は逆向き循環を避ける論理pointer。Repositoryがsame Workflow/Job relationをtransaction内検証する。

Enum/check:

```text
executor=internal|external_llm|human|reusable
status=queued|running|waiting_external|waiting_review|waiting_child|completed
conclusion=NULL|success|failure|cancelled|skipped|blocked
completed <=> conclusion non-NULL
internal: action_id/version/runs_on non-empty, uses_ref NULL
external: action/version/runs_on/uses_ref NULL
human: action/version/validator/runs_on/uses_ref NULL
reusable: action/version/validator/runs_on NULL, uses_ref non-empty
validator pair both NULL or both non-empty
timeout: internal only, NULL or finite>0
current_attempt_no=0 -> current_attempt_id NULL
current_attempt_no>0 -> current_attempt_id non-empty
```

Indexes:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
CREATE INDEX ix_job_ready_internal
ON job_runs(runs_on,status,retry_not_before,priority DESC,ready_at ASC,id ASC)
WHERE executor='internal' AND status='queued';
CREATE INDEX ix_job_by_workflow
ON job_runs(workflow_run_id,source_order ASC,id ASC);
CREATE INDEX ix_job_retry_due
ON job_runs(retry_not_before)
WHERE retry_not_before IS NOT NULL AND status='queued';
CREATE INDEX ix_job_reuse_pending
ON job_runs(workflow_run_id,reuse_check_pending)
WHERE reuse_check_pending=1;
```

## 9. `job_attempts`

```text
id TEXT PRIMARY KEY
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_no INTEGER NOT NULL CHECK(attempt_no>=1)
status TEXT NOT NULL
conclusion TEXT NULL
runner_id TEXT NULL
runner_instance_id TEXT NULL
runtime_instance_id TEXT NULL
input_json TEXT NOT NULL
output_storage_kind TEXT NULL
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
reuse_eligible INTEGER NULL CHECK(reuse_eligible IN(0,1))
reuse_context_json TEXT NULL
reuse_key TEXT NULL
failure_json TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
UNIQUE(job_run_id,attempt_no)
```

```text
status=running|waiting_external|waiting_review|waiting_child|completed
conclusion=NULL|success|failure|cancelled
completed <=> conclusion non-NULL
```

Pre-Attempt failureはAttempt row無し。

```sql
CREATE INDEX ix_attempts_job ON job_attempts(job_run_id,attempt_no DESC);
```

## 10. `job_steps`

```text
id TEXT PRIMARY KEY
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
sequence INTEGER NOT NULL CHECK(sequence>=1)
name TEXT NOT NULL CHECK(length(name)>0)
status TEXT NOT NULL
conclusion TEXT NULL
metadata_json TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
UNIQUE(attempt_id,sequence)
```

```text
status=running|completed
conclusion=NULL|success|failure|cancelled|incomplete
completed <=> conclusion non-NULL
```

Open StepはAttempt terminal/Recoveryで`incomplete`へ閉じる。

## 11. `dynamic_expansions`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
template_id TEXT NOT NULL CHECK(length(template_id)>0)
parent_generated_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
outcome TEXT NOT NULL
source_snapshot_json TEXT NULL
source_digest TEXT NULL
generated_count INTEGER NOT NULL DEFAULT 0 CHECK(generated_count>=0)
failure_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

```text
outcome=pending|expanded|skipped|failed|cancelled
pending -> completed_at NULL
terminal -> completed_at non-NULL
expanded -> source snapshot/digest non-NULL
skipped|cancelled -> generated_count=0
failed -> failure_json non-NULL
```

```sql
CREATE UNIQUE INDEX uq_dynamic_expansion_root
ON dynamic_expansions(workflow_run_id,template_id)
WHERE parent_generated_job_run_id IS NULL;
CREATE UNIQUE INDEX uq_dynamic_expansion_nested
ON dynamic_expansions(workflow_run_id,template_id,parent_generated_job_run_id)
WHERE parent_generated_job_run_id IS NOT NULL;
```

`if=false`=`skipped`、`foreach=[]`=`expanded/count0`。Generated jobs + snapshot + outcomeは1 transaction。Zero-parent direct propagationでfake rowを作らない。

## 12. `reusable_bindings`

```text
id TEXT PRIMARY KEY
parent_workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
parent_job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
workflow_ref_original TEXT NOT NULL CHECK(length(workflow_ref_original)>0)
child_workflow_id TEXT NOT NULL CHECK(length(child_workflow_id)>0)
child_workflow_version INTEGER NOT NULL CHECK(child_workflow_version>=1)
child_definition_yaml TEXT NOT NULL
child_definition_json TEXT NOT NULL
child_definition_hash TEXT NOT NULL CHECK(length(child_definition_hash)>0)
child_action_versions_json TEXT NOT NULL
child_validator_versions_json TEXT NOT NULL
created_at TEXT NOT NULL
UNIQUE(parent_job_run_id)
```

## 13. `workflow_state`

```text
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
revision INTEGER NOT NULL CHECK(revision>=1)
value_json TEXT NOT NULL
updated_at TEXT NOT NULL
PRIMARY KEY(workflow_run_id,name)
```

## 14. `workflow_state_history`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
revision INTEGER NOT NULL CHECK(revision>=1)
old_value_json TEXT NULL
new_value_json TEXT NOT NULL
job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
step_id TEXT NULL REFERENCES job_steps(id) ON DELETE NO ACTION
actor_json TEXT NULL
created_at TEXT NOT NULL
UNIQUE(workflow_run_id,name,revision)
```

`state.set`: current read -> history insert -> current upsertを1 transaction。

```sql
CREATE INDEX ix_state_history
ON workflow_state_history(workflow_run_id,name,revision DESC);
```

## 15. `artifacts`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
storage_kind TEXT NOT NULL
store_key TEXT NULL
external_uri TEXT NULL
media_type TEXT NULL
size_bytes INTEGER NULL CHECK(size_bytes IS NULL OR size_bytes>=0)
digest TEXT NULL
metadata_json TEXT NULL
created_at TEXT NOT NULL
data_deleted_at TEXT NULL
metadata_deleted_at TEXT NULL
```

```text
storage_kind=managed|external
managed -> store_key non-empty, external_uri NULL
external -> store_key NULL, external_uri non-empty
data_deleted_at only managed
metadata_deleted_at -> current resolution外
```

Same Attempt/name複数generation可。Current候補は`created_at DESC,id DESC`。

```sql
CREATE INDEX ix_artifact_current
ON artifacts(job_run_id,attempt_id,name,created_at DESC,id DESC);
CREATE INDEX ix_artifact_workflow
ON artifacts(workflow_run_id,created_at ASC,id ASC);
```

Cross-run ArtifactRefはRetention hold無し。

## 16. `events`

```text
id TEXT PRIMARY KEY
type TEXT NOT NULL CHECK(length(type)>0)
version INTEGER NOT NULL DEFAULT 1 CHECK(version>=1)
workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
runner_id TEXT NULL
actor_type TEXT NULL
actor_id TEXT NULL
source TEXT NULL
request_id TEXT NULL
payload_json TEXT NOT NULL
created_at TEXT NOT NULL
```

Retention audit Event=`retention_deleted|retention_orphan_cleaned`はRun/Job/Attempt ID NULL、通常event-daysから除外し無期限保持。

```sql
CREATE INDEX ix_events_workflow ON events(workflow_run_id,created_at ASC,id ASC);
CREATE INDEX ix_events_type_time ON events(type,created_at ASC,id ASC);
```

## 17. `execution_logs`

```text
id TEXT PRIMARY KEY
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
relative_path TEXT NOT NULL CHECK(length(relative_path)>0)
size_bytes INTEGER NOT NULL DEFAULT 0 CHECK(size_bytes>=0)
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
closed_at TEXT NULL
deleted_at TEXT NULL
UNIQUE(attempt_id)
```

PathはCore生成data-root相対。Retention後`deleted_at`で空Logと区別。

## 18. `runners`

```text
runner_instance_id TEXT PRIMARY KEY
runtime_instance_id TEXT NOT NULL
runner_id TEXT NOT NULL
pool_name TEXT NOT NULL CHECK(length(pool_name)>0)
pid INTEGER NOT NULL CHECK(pid>=0)
status TEXT NOT NULL
current_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
current_attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
started_at TEXT NOT NULL
last_heartbeat_at TEXT NULL
main_loop_tick_at TEXT NULL
stopped_at TEXT NULL
updated_at TEXT NOT NULL
```

Status=`starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed`。

Invariant:

- `running` -> current Job/Attempt non-NULL
- `idle|stopped|lost|restart_suppressed` -> current Job/Attempt NULL
- Attempt terminal/recovery時にRunner current pointersを同logical operationでclear

```sql
CREATE UNIQUE INDEX uq_runner_current_slot
ON runners(runtime_instance_id,pool_name,runner_id)
WHERE status IN('starting','idle','claiming','running','stopping');
CREATE INDEX ix_runners_liveness
ON runners(runtime_instance_id,status,last_heartbeat_at);
CREATE INDEX ix_runners_pool
ON runners(runtime_instance_id,pool_name,runner_id);
```

## 19. `runner_restarts`

```text
id TEXT PRIMARY KEY
runtime_instance_id TEXT NOT NULL
pool_name TEXT NOT NULL
runner_id TEXT NOT NULL
exited_runner_instance_id TEXT NULL
exit_code INTEGER NULL
reason TEXT NOT NULL
restart_ordinal INTEGER NOT NULL CHECK(restart_ordinal>=1)
scheduled_for TEXT NULL
started_runner_instance_id TEXT NULL
suppressed INTEGER NOT NULL DEFAULT 0 CHECK(suppressed IN(0,1))
created_at TEXT NOT NULL
```

```sql
CREATE INDEX ix_runner_restart_window
ON runner_restarts(runtime_instance_id,pool_name,runner_id,created_at DESC);
```

## 20. `external_tasks`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
status TEXT NOT NULL
available_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(attempt_id)
```

Status=`available|leased|completed|cancelled`。

```text
completed -> completed_at non-NULL
cancelled -> cancelled_at non-NULL
available|leased -> terminal timestamps NULL
```

```sql
CREATE INDEX ix_external_task_available
ON external_tasks(status,available_at,id)
WHERE status='available';
```

## 21. `external_leases`

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL REFERENCES external_tasks(id) ON DELETE NO ACTION
claimant_key TEXT NOT NULL CHECK(length(claimant_key)>0)
status TEXT NOT NULL
claimed_at TEXT NOT NULL
expires_at TEXT NOT NULL
released_at TEXT NULL
invalidated_at TEXT NULL
created_at TEXT NOT NULL
```

ID=public lease_id。Status=`active|expired|released|invalidated`。

```text
active -> released_at/invalidated_at NULL
released -> released_at non-NULL
invalidated -> invalidated_at non-NULL
```

```sql
CREATE UNIQUE INDEX uq_external_active_lease
ON external_leases(task_id)
WHERE status='active';
CREATE INDEX ix_external_lease_due
ON external_leases(expires_at)
WHERE status='active';
```

Heartbeat/renew/transfer column無し。

## 22. `human_reviews`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
status TEXT NOT NULL
outcome TEXT NULL
comment TEXT NULL
actor_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
UNIQUE(attempt_id)
```

```text
status=pending|completed|cancelled
outcome=NULL|approve|reject
pending -> outcome/completed_at/cancelled_at NULL
completed -> outcome and completed_at non-NULL
cancelled -> outcome NULL and cancelled_at non-NULL
```

Completed outcome rewrite不可。

```sql
CREATE INDEX ix_reviews_list
ON human_reviews(status,created_at ASC,id ASC);
```

## 23. `idempotency_records`

MVPでは**completed resultだけを永続化**し、別transactionの`reserved` rowを作らない。

```text
scope TEXT NOT NULL
operation TEXT NOT NULL
request_key TEXT NOT NULL
request_hash TEXT NOT NULL
result_json TEXT NOT NULL
adapter_meta_json TEXT NULL
created_at TEXT NOT NULL
expires_at TEXT NOT NULL
PRIMARY KEY(scope,operation,request_key)
```

正規flow:

1. optional fast readで既存key確認
2. filesystem prepareが必要ならtempだけprepare（side effectをcurrent stateへ公開しない）
3. `BEGIN IMMEDIATE`
4. keyを**再確認**
5. unexpired same hash -> existing result replay、今回tempはorphan cleanup
6. unexpired different hash -> idempotency_conflict、今回temp cleanup
7. expired row -> delete/replace可能
8. domain current stateを再検証
9. side effect + completed result rowをsame transactionでcommit

これにより同一key concurrent requestが両方side effectをcommitしない。Process crashで`reserved` keyが永続的に残る問題も作らない。

Default TTL24h。Result/adapter metadata SecretGuard。

```sql
CREATE INDEX ix_idempotency_expiry
ON idempotency_records(expires_at);
```

## 24. Concurrency transaction

Dedicated table無し。

Workflow start/slot releaseは`BEGIN IMMEDIATE`でsame BINARY groupのactive holderを再count。

Active holder=`queued|running|paused`かつ`wait_reason != concurrency`。

Waiting release:

```text
priority DESC
created_at ASC
id ASC
```

## 25. Result Reuse persistence

Successful Attempt terminal時に`reuse_context_json/reuse_key/reuse_eligible`を確定。

```text
workflow_run_id/job_key/definition_hash
persistent_input_digest
direct_upstream_artifacts
executor_identity
validator_identity
ineligibility_reason optional
```

- state_get -> ineligible
- persistent Input外Artifact materialize -> ineligible

Output commit + reuse metadata + Job success same logical terminal operation。

Manual Retry reopen:

1. failed Attempt/Input + version availability
2. run_attempt++/Run reopen
3. target queued
4. blocked/skipped reset
5. success descendant reuse_check_pending
6. Event/idempotency

## 26. Retention / delete order

Policy=`workflow_runs.retention_policy_json`。

- Run: completed_at、non-terminal delete無し
- Log: close/completed time、running delete無し
- normal Event: created_at、retention audit除外
- Artifact data/metadata: created_at + owner non-terminal guard
- Output Payload: run-historyと共にdelete
- Cross-run ArtifactRefはRetention hold無し
- External Artifact実体delete無し

### 26.1 Recursive Child deletion

Child Workflow RunはParent Job/Attemptをlogical pointerで参照するため、Run history deletionは**call_depth DESC**でdescendant Child subtreeを先に削除する。

1 Run subtree内の代表順:

1. descendant Child Run subtree recursively
2. external_leases
3. human_reviews
4. execution log file + execution_logs row
5. managed Artifact data + artifacts row / external artifact metadata
6. workflow_state_history / workflow_state
7. normal events
8. job_steps
9. external_tasks
10. dynamic_expansions
11. job_attempts
12. reusable_bindings
13. job_runs
14. Workflow Output blob
15. workflow_runs row

Runner rowsがcompleted Run Job/Attemptをcurrent pointerで保持している場合はinvariant violationとしてRecovery/repairでpointer clear後にRetentionする。FK無効化で強制削除しない。

System-level retention audit Eventはowner FK無しで削除前/同logical operationに残す。

## 27. Transaction boundaries

1 DB transaction all-or-nothing:

- Workflow Run start DB rows + idempotency result
- internal Job claim + Attempt + Runner ownership
- Dynamic expansion DB registration
- state current + history
- External Lease claim
- External submit terminal DB state
- Human review first-wins
- Manual Retry reopen
- concurrency holder decision

Payload/Artifact filesystemはtemp/finalize + DB transaction + orphan cleanup。Distributed transaction/exactly-once無し。

## 28. Migration / schema verification

Migration=`migrations/NNN_name.sql`。

Startup:

1. schema_migrations read
2. duplicate/gap/unknown future reject
3. pending migrations sequentially apply
4. expected final version verify

Testは`sqlite_master`/PRAGMAでtable/column/index/FK/checkを実DB確認。

## 29. 受入条件

1. **全18 table** exact column/schema presence
2. enum/check/output payload invariants
3. actual FK vs logical pointer contract
4. one-running internal partial unique
5. Dynamic root/nested partial unique + skip/empty persistence
6. Reusable binding/Child uniqueness
7. state current/history atomic
8. Artifact current/retention/cross-run no-pin
9. Event/log schema + retention audit
10. Runner pointer/liveness/restart invariants
11. External Task/active Lease/no renew fields
12. Human Review immutability
13. idempotency no-reserved/concurrent recheck/TTL replay
14. deadline indexes
15. concurrency transaction
16. Result Reuse persistence
17. recursive Child-first Retention
18. migration future/gap reject
19. Payload/Artifact crash consistency
