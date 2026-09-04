# 08. Persistence 詳細設計

- Status: Draft v2.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- Canonical JSON: `01` の `jobrunner.canonical-json.v1`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Input snapshot、Dynamic expansion、Result Reuse、deadline、Retention、Migration、Recovery、Idempotencyを実装可能な正規契約として定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ persistent Job Input/Secret binding metadata
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

- 必要時`BEGIN IMMEDIATE`
- 長transaction禁止
- FK原則`ON DELETE NO ACTION`
- implicit cascade無し
- Text identity=BINARY semantics
- Timestamp=UTC RFC3339 TEXT
- ID=type prefix + UUID4 hex TEXT
- INTEGER外部入力=signed64
- JSON/digest=canonical-json-v1
- boolean=INTEGER 0|1

RetentionはFKを無効化せず明示順序。

## 4. Canonical table set

MVP tableは**18個**。

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

Workflow Definition履歴tableは作らない。実行済みDefinitionは`workflow_runs` snapshot。

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
inline: kind=inline,json non-NULL,blob NULL,size>=0,digest non-empty
blob: kind=blob,json NULL,blob non-empty,size>=0,digest non-empty
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
effective_settings_json TEXT NOT NULL
retention_policy_json TEXT NOT NULL
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
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

Logical pointers=`parent_job_run_id/parent_attempt_id/reusable_binding_id`。Repositoryがsame relationをtransaction内検証する。

Enums/invariants:

```text
status=queued|running|paused|completed
conclusion=NULL|success|failure|cancelled
completed <=> conclusion non-NULL
source_identity NULL or non-empty
concurrency_group NULL <=> max/on_limit NULL
concurrency_max_runs NULL or >=1
concurrency_on_limit NULL|queue|reject
root: parent_workflow_run_id NULL,depth=0,root_workflow_run_id=id
child: parent ids/binding non-empty,depth>=1
```

`effective_settings_json` exact keys:

```text
max_dynamic_jobs
external_lease_minutes
external_on_lease_expiry
output_inline_threshold_bytes
execution_log_level
workflow_progress_mode
job_progress_mode
```

Retention policy exact keysは`01`の5項目。

Indexes:

```sql
CREATE INDEX ix_workflow_runs_list ON workflow_runs(created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_by_workflow ON workflow_runs(workflow_id,created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_status ON workflow_runs(status,priority DESC,created_at ASC,id ASC);
CREATE INDEX ix_workflow_runs_concurrency ON workflow_runs(concurrency_group,status)
  WHERE concurrency_group IS NOT NULL AND status!='completed';
CREATE UNIQUE INDEX uq_child_run_per_parent_attempt ON workflow_runs(parent_attempt_id)
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
order_rank INTEGER NULL CHECK(order_rank IS NULL OR order_rank>=0)
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
ready_at TEXT NULL
pending_input_json TEXT NULL
pending_secret_bindings_json TEXT NULL
pending_input_digest TEXT NULL
current_attempt_no INTEGER NOT NULL DEFAULT 0 CHECK(current_attempt_no>=0)
current_attempt_id TEXT NULL
reuse_check_pending INTEGER NOT NULL DEFAULT 0 CHECK(reuse_check_pending IN(0,1))
current_failure_json TEXT NULL
progress_mode TEXT NOT NULL
progress_current REAL NULL
progress_total REAL NULL
progress_message TEXT NULL
progress_unit TEXT NULL
progress_updated_at TEXT NULL
queued_at TEXT NULL
started_at TEXT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(workflow_run_id,job_key)
```

Logical pointers=`dynamic_expansion_id/reusable_binding_id/current_attempt_id`。

Enums:

```text
executor=internal|external_llm|human|reusable
status=queued|running|waiting_external|waiting_review|waiting_child|completed
conclusion=NULL|success|failure|cancelled|skipped|blocked
progress_mode=auto|explicit|none
completed <=> conclusion non-NULL
```

Executor invariants:

```text
internal: action/version/runs_on non-empty,uses_ref NULL
external: action/version/runs_on/uses_ref NULL
human: action/version/validator/runs_on/uses_ref NULL
reusable: action/version/validator/runs_on NULL,uses_ref non-empty
validator pair both NULL or both non-empty
timeout only internal,finite>0
```

Readiness/Input invariant:

```text
internal + status=queued + ready_at non-NULL
  -> pending_input_json/pending_secret_bindings_json/pending_input_digest non-NULL
internal + status=running
  -> current_attempt_id non-NULL
non-internal
  -> pending_* NULL
```

`pending_secret_bindings_json` is canonical array from`02`; no Secret value。

Progress numeric invariantは`09`。

Indexes:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';

CREATE INDEX ix_job_ready_internal
ON job_runs(runs_on,status,retry_not_before,priority DESC,ready_at ASC,id ASC)
WHERE executor='internal' AND status='queued' AND ready_at IS NOT NULL;

CREATE INDEX ix_job_by_workflow ON job_runs(workflow_run_id,source_order ASC,id ASC);
CREATE INDEX ix_job_retry_due ON job_runs(retry_not_before)
WHERE retry_not_before IS NOT NULL AND status='queued';
CREATE INDEX ix_job_reuse_pending ON job_runs(workflow_run_id,reuse_check_pending)
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
secret_bindings_json TEXT NOT NULL
input_digest TEXT NOT NULL CHECK(length(input_digest)>0)
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
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

Pre-Attempt failureはrow無し。

Internal claimはJob pending snapshotをexact copy。External/Human/Reusable activationは直接Attemptへsnapshot。Secret禁止executorはbindings=`[]`。

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

`status=running|completed`, conclusion=`NULL|success|failure|cancelled|incomplete`。

## 11. `dynamic_expansions`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
template_id TEXT NOT NULL CHECK(length(template_id)>0)
parent_generated_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
outcome TEXT NOT NULL
source_snapshot_json TEXT NULL
source_digest TEXT NULL
expansion_digest TEXT NULL
generated_count INTEGER NOT NULL DEFAULT 0 CHECK(generated_count>=0)
failure_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

```text
outcome=pending|expanded|skipped|failed|cancelled
pending -> completed_at NULL
terminal -> completed_at non-NULL
expanded -> source_snapshot/source_digest/expansion_digest non-NULL
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

`if=false`=skipped、`foreach=[]`=expanded/count0。Generated jobs + source + digest + outcomeは1 transaction。

Whole skipped group reactivation時はgenerated0のskipped rowをreset/remove可能。System Eventでprior skip履歴を残す。Expanded rowはsame Runでgenerated setを書き換えない。

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
child_definition_hash TEXT NOT NULL
child_action_versions_json TEXT NOT NULL
child_validator_versions_json TEXT NOT NULL
created_at TEXT NOT NULL
UNIQUE(parent_job_run_id)
```

## 13. `workflow_state` / `workflow_state_history`

Current:

```text
workflow_run_id TEXT NOT NULL FK
name TEXT NOT NULL
revision INTEGER NOT NULL CHECK(revision>=1)
value_json TEXT NOT NULL
updated_at TEXT NOT NULL
PRIMARY KEY(workflow_run_id,name)
```

History:

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL FK
name TEXT NOT NULL
revision INTEGER NOT NULL CHECK(revision>=1)
old_value_json TEXT NULL
new_value_json TEXT NOT NULL
job_run_id/attempt_id/step_id nullable FK
actor_json TEXT NULL
created_at TEXT NOT NULL
UNIQUE(workflow_run_id,name,revision)
```

`state.set`=history insert + current upsert same transaction。Last-write-wins。

## 14. `artifacts`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL FK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK
name TEXT NOT NULL
storage_kind TEXT NOT NULL
store_key TEXT NULL
external_uri TEXT NULL
media_type TEXT NULL
size_bytes INTEGER NULL
digest TEXT NULL
metadata_json TEXT NULL
created_at TEXT NOT NULL
data_deleted_at TEXT NULL
metadata_deleted_at TEXT NULL
```

```text
storage_kind=managed|external
managed -> store_key non-empty,external_uri NULL
external -> store_key NULL,external_uri non-empty
data_deleted_at only managed
metadata_deleted_at -> current resolution外
```

Same Attempt/name複数generation可。Current=`created_at DESC,id DESC`。Cross-run ArtifactRef retention pin無し。

## 15. `events`

```text
id TEXT PRIMARY KEY
type TEXT NOT NULL
version INTEGER NOT NULL DEFAULT 1 CHECK(version>=1)
workflow_run_id/job_run_id/attempt_id nullable FK
runner_id TEXT NULL
actor_type/actor_id/source/request_id TEXT NULL
payload_json TEXT NOT NULL
created_at TEXT NOT NULL
```

System retention audit=`retention_deleted|retention_orphan_cleaned`はowner IDs NULL、通常event retention外・MVP無期限。

Indexes by workflow/time and type/time。

## 16. `execution_logs`

**全executorのAttempt**に最大1 logical Execution Log。

```text
id TEXT PRIMARY KEY
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
relative_path TEXT NOT NULL
size_bytes INTEGER NOT NULL DEFAULT 0 CHECK(size_bytes>=0)
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
closed_at TEXT NULL
deleted_at TEXT NULL
UNIQUE(attempt_id)
```

Internalはclaim時、External/Human/Reusableはactivation時に作成する。PathはCore生成。Retention後`deleted_at`。

## 17. `runners`

```text
runner_instance_id TEXT PRIMARY KEY
runtime_instance_id TEXT NOT NULL
runner_id TEXT NOT NULL
pool_name TEXT NOT NULL
pid INTEGER NOT NULL CHECK(pid>=0)
status TEXT NOT NULL
current_job_run_id TEXT NULL FK
current_attempt_id TEXT NULL FK
started_at TEXT NOT NULL
last_heartbeat_at TEXT NULL
main_loop_tick_at TEXT NULL
stopped_at TEXT NULL
updated_at TEXT NOT NULL
```

Status=`starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed`。

Running -> current pointers non-null。Idle/terminal runner states -> current pointers null。

Unique current slot=`runtime_instance_id,pool_name,runner_id` for active statuses。

## 18. `runner_restarts`

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

Window query index=`runtime,pool,runner,created_at DESC`。

## 19. `external_tasks`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL FK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK
status TEXT NOT NULL
lease_seconds REAL NOT NULL
on_lease_expiry TEXT NOT NULL
available_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(attempt_id)
```

```text
status=available|leased|completed|cancelled
lease_seconds finite>0
on_lease_expiry=requeue|fail
completed -> completed_at non-NULL
cancelled -> cancelled_at non-NULL
available|leased -> terminal timestamps NULL
```

Task configはactivation snapshot。Requeueで変更しない。

Available candidate index=`status,available_at,id`。

## 20. `external_leases`

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL FK
claimant_key TEXT NOT NULL
status TEXT NOT NULL
claimed_at TEXT NOT NULL
expires_at TEXT NOT NULL
released_at TEXT NULL
invalidated_at TEXT NULL
created_at TEXT NOT NULL
```

Status=`active|expired|released|invalidated`。

Active one/task partial unique。Due index=`expires_at WHERE active`。

Heartbeat/renew/transfer column無し。

## 21. `human_reviews`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL FK
job_run_id TEXT NOT NULL FK
attempt_id TEXT NOT NULL FK
status TEXT NOT NULL
outcome TEXT NULL
comment TEXT NULL
actor_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
UNIQUE(attempt_id)
```

Status=`pending|completed|cancelled`。Outcome=`NULL|approve|reject`。Completed outcome rewrite不可。

## 22. `idempotency_records`

Completed resultだけ永続化し`reserved` row無し。

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

### 22.1 `request_hash`

```text
SHA-256(canonical-json-v1(Service request model excluding request_id and transport-only fields))
```

HTTP pathからinjectされたresource ID、`claim_next`、result/artifact metadata等のService意味fieldは含む。Actor/AccessScopeはrequest hashではなくscopeへ入れる。

### 22.2 `scope`

Coreは次のnon-secret canonical structureをhashしてopaque scope keyを作る。

```text
namespace
operation_resource_key
actor_principal_key
access_scope
```

`actor_principal_key`はAdapter/parentがActorContext/client identityから安定して供給する。AccessScopeはcanonical-json-v1 compatibleであること。

### 22.3 flow

1. optional fast read
2. filesystem prepareはtemp/final prepareまで
3. `BEGIN IMMEDIATE`
4. key再確認
5. unexpired same hash -> replay
6. unexpired different hash -> conflict
7. expired -> replace可
8. domain current state再検証
9. side effect + completed result row同transaction commit

No DB reservation row。SQLite write serialization + commit-time recheckがconcurrent duplicate commitを防ぐ。

TTL=`System idempotency_ttl_hours`, canonical24h。

## 23. Concurrency transaction

Dedicated table無し。

Workflow start/slot releaseは`BEGIN IMMEDIATE`でsame BINARY group active holderを再count。

Active holder=`queued|running|paused`かつ`wait_reason != concurrency`。

Waiting release=`priority DESC,created_at ASC,id ASC`。

## 24. Result Reuse persistence

Successful Attempt terminal時:

```text
reuse_context_json
reuse_key
reuse_eligible
```

Contextは`03`。

Manual Retry successful descendantはcurrent `if`/expected input/artifact/validationを再計算し、stored Inputだけ比較しない。

Dynamic successful expansionは`expansion_digest` exact再検証後にgenerated Job reuseへ進む。

## 25. Retention / delete order

Policy=`workflow_runs.retention_policy_json`。

- Run: completed_at、non-terminal delete無し
- Log: close time、running delete無し
- normal Event: created_at、retention audit除外
- Artifact data/metadata: created_at + owner non-terminal guard
- Output Payload: run-historyと共にdelete
- Cross-run ArtifactRef retention hold無し
- External Artifact実体delete無し

Child subtreeは`call_depth DESC`で先に削除。

1 Runの代表順:

1. descendant Child subtree
2. external_leases
3. human_reviews
4. execution log file/row
5. Artifact data/row
6. state history/current
7. normal events
8. job_steps
9. external_tasks
10. dynamic_expansions
11. job_attempts
12. reusable_bindings
13. job_runs
14. Workflow Output blob
15. workflow_runs

Runner current pointerが残る場合はrepair後。FK無効化禁止。

## 26. Transaction boundaries

1 DB transaction all-or-nothing:

- Workflow Run start + idempotency result
- internal claim + Attempt + Runner ownership
- Dynamic expansion registration
- state current + history
- External Lease claim + idempotency result
- External submit terminal + **optional claim_next Lease + full idempotency result**
- Human review first-wins + idempotency result
- Manual Retry reopen + idempotency result
- concurrency holder decision

Payload/Artifact filesystemはprepare/finalize + DB transaction + orphan cleanup。Distributed exactly-once無し。

## 27. Migration / schema verification

Migration=`migrations/NNN_name.sql`。

Startup:

1. `schema_migrations` read
2. duplicate/gap/unknown future reject
3. pending sequential apply
4. expected final version verify

Testは`sqlite_master`/PRAGMAでtable/column/index/FK/checkを実DB検証。

## 28. 受入条件

1. 全18 table exact schema
2. effective_settings/retention snapshot
3. internal queued ready/pending Input invariants
4. Attempt secret bindings/input digest
5. Runner claim exact snapshot copy
6. one-running internal partial unique
7. Dynamic expansion digest/unique/skip-empty
8. Reusable binding/Child unique
9. State atomic
10. Artifact/log/common executor schema
11. Runner liveness/restart invariants
12. External Task lease config snapshot
13. active Lease/no renew
14. Human Review immutability
15. idempotency request hash/scope/no-reserved/recheck/TTL
16. submit+claim_next+idempotency single transaction
17. concurrency transaction
18. Result Reuse current-context precheck
19. Child-first Retention
20. migration future/gap reject
21. Payload/Artifact crash consistency
