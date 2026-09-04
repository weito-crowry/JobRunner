# 08. Persistence 詳細設計

- Status: Draft v0.9
- 対象: MVP
- 上位仕様: `docs/design.md`
- Canonical JSON: `01-workflow-definition.md` の `jobrunner.canonical-json.v1`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Action/Validator snapshot、Result Reuse、deadline、Retention、Migration、Recovery、Idempotencyを定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ Payload metadata
├─ Artifact metadata
├─ Action/Validator identity
└─ Reuse/idempotency/retention metadata

PayloadStore
└─ large JSON Output blob

ArtifactStore
└─ explicitly managed artifact data
```

MVP standard=`sqlite3`, `LocalHybridPayloadStore`, `LocalArtifactStore`。

## 3. SQLite設定

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000
PRAGMA journal_mode=WAL
```

競合writeは必要時`BEGIN IMMEDIATE`。長transaction禁止。

Text identityはBINARY semantics。Concurrency group case-sensitive。

## 4. ID / time / JSON / numeric

ID=type prefix + UUID4 hex。UTC RFC3339。

JSON persistent/digest対象は`jobrunner.canonical-json.v1`。別serializer禁止。

SQLite INTEGER外部入力=signed64範囲。必須identity=non-empty TEXT。

## 5. Foreign Key policy

FK原則`ON DELETE NO ACTION`。Implicit cascade無し。

Retention Serviceがleaf -> attempt-owned -> job-owned -> child runs -> root run順に明示削除。FK無効化禁止。

## 6. Tables

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

Deadline専用table無し。`retry_not_before` / `expires_at`がSource of Truth。

## 7. Output payload columns

```text
output_storage_kind  NULL|inline|blob
output_json          NULL|TEXT
output_blob_key      NULL|TEXT
output_size_bytes    NULL|INTEGER >=0
output_digest        NULL|TEXT
```

none=all null、inline=json only、blob=blob key only、Outputありならsize/digest必須。

## 8. PayloadStore crash consistency

1. validation/SecretGuard
2. canonical-json-v1 bytes + SHA-256
3. threshold
4. blob temp write -> atomic rename
5. DB terminal transaction

DB commit前orphan blobはmaintenance cleanup。Readはexistence/size/digest verify。

## 9. `workflow_runs`

主要:

```text
id TEXT PK
workflow_id TEXT NOT NULL
workflow_version INTEGER NOT NULL CHECK >=1
status/conclusion/priority/run_attempt/wait_reason
pause_requested/cancel_requested

definition_yaml/definition_json/definition_hash
input_json
action_versions_json TEXT NOT NULL
validator_versions_json TEXT NOT NULL
retention_policy_json TEXT NOT NULL
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
failure_json/source_identity

concurrency_group TEXT COLLATE BINARY NULL
concurrency_max_runs/concurrency_on_limit
actor_context_json/access_scope_json
parent_workflow_run_id/parent_job_run_id/parent_attempt_id/root_workflow_run_id/call_depth/reusable_binding_id
created_at/started_at/completed_at/updated_at
```

`retention_policy_json` exact shape:

```text
run_history_days: integer>=1|null
execution_logs_days: integer>=1|null
event_days: integer>=1|null
artifact_metadata_days: integer>=1|null
managed_artifact_data_days: integer>=1|null
```

Run start時effective policyをcanonical JSONでsnapshot。後からSystem/Workflow source設定が変わっても既存Run policyを暗黙変更しない。

Checks:

```text
status=queued|running|paused|completed
conclusion=NULL|success|failure|cancelled
completed <=> conclusion non-NULL
concurrency_group NULL or length>0
source_identity NULL or length>0
```

## 10. `job_runs`

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
job_key/job_template_key/dynamic_key/parent_job_run_id/dynamic_expansion_id
item_snapshot_json/iteration_context_json/source_order/order_rank
executor TEXT NOT NULL
action_id/action_version NULL
validator_id/validator_version NULL
uses_ref/reusable_binding_id/runs_on
status/conclusion/terminal_reason
priority/continue_on_error
timeout_seconds/retry_policy_json/retry_not_before
current_attempt_no/current_attempt_id
reuse_check_pending INTEGER NOT NULL DEFAULT 0 CHECK IN(0,1)
current_failure_json
ready_at/queued_at/started_at/completed_at/created_at/updated_at
```

Executor/status/conclusion checksは`01/03`とexact一致。

Validator pair both null or both non-empty。Human/Reusable validator無し。Internal timeout finite>0。

`UNIQUE(workflow_run_id,job_key)`。

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
```

## 11. `job_attempts`

```text
id TEXT PK
job_run_id TEXT NOT NULL FK
attempt_no INTEGER NOT NULL CHECK >=1
status/conclusion
runner_id/runner_instance_id/runtime_instance_id
input_json TEXT NOT NULL
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
reuse_eligible INTEGER NULL CHECK IN(0,1)
reuse_context_json/reuse_key/failure_json
started_at/completed_at/created_at
UNIQUE(job_run_id,attempt_no)
```

Attempt status=`running|waiting_external|waiting_review|waiting_child|completed`。

Pre-Attempt failureはJob Runで表現しInput snapshot無し。

## 12. Reuse context

Successful terminal時:

```text
workflow_run_id/job_key/definition_hash
persistent_input_digest
direct_upstream_artifacts
executor_identity
validator_identity
ineligibility_reason optional
```

canonical-json-v1 SHA-256。

state_get/undeclared Artifact materialize -> ineligible。

Output commit + reuse metadata + Job success same logical terminal operation。

## 13. Manual Retry reopen

1. failed Attempt/input存在
2. 無し -> retry_input_unavailable no change
3. Action/Validator/Binding version availability
4. run_attempt++/Run reopen
5. target retry waiting
6. blocked/skipped reset
7. successful reuse_check_pending
8. Event/idempotency

## 14. Deadline indexes

```sql
CREATE INDEX ix_job_retry_due
ON job_runs(retry_not_before)
WHERE retry_not_before IS NOT NULL AND status='queued';

CREATE INDEX ix_external_lease_due
ON external_leases(expires_at)
WHERE status='active';
```

Conditional transaction、due二重処理は副作用化しない。

## 15. Dynamic / Reusable uniqueness

Dynamic root/nested expansion partial unique。Full job_key fixed length CHECK無し。

Reusable binding one/parent_job_run_id。BindingにChild Action+Validator versions。

Child Run one/parent_attempt_id。

## 16. State

Current + append-only history same transaction。SecretGuard対象。

## 17. Artifact metadata / Store

```text
id/workflow_run_id/job_run_id/attempt_id/name
storage_kind=managed|external
store_key/external_uri
media_type/size/digest/metadata
created_at/data_deleted_at/metadata_deleted_at
```

Managed put=temp copy/Secret scan/digest/atomic finalize/metadata Event。DB failure orphan cleanup。

## 18. Event persistence / retention audit

通常Eventは`workflow_run_id`等をnullable参照として持つが、Run row削除後もRetention実施事実を残すため、**system-level retention audit Eventはworkflow_run_id=NULL** としpayloadへdeleted resource type/opaque ID/countを記録する。

Run削除前にsystem-level `retention_deleted` Eventをcommitし、対象RunにFK依存させない。

SecretGuard対象。

## 19. External / Human uniqueness

- one Task/Attempt
- one active Lease/Task
- one Review/Attempt

## 20. Idempotency

```text
scope/operation/request_key PK
request_hash/result_json/adapter_meta_json/status/created_at/expires_at
```

Default TTL24h。Scope=namespace+resource+AccessScope+Actor/client principal。

TTL内replay/conflict、expired replace可。Result/adapter metadata SecretGuard対象。

Side effect + idempotency result same transaction。

## 21. Concurrency

Start/slot release `BEGIN IMMEDIATE` holder recount。Group BINARY case-sensitive。

## 22. Migration

`migrations/NNN_name.sql` + `schema_migrations`。Ordered once、fail-closed、unknown future version reject。

## 23. Retention deletion rules

Policy source=`workflow_runs.retention_policy_json`。

- Run age base: `completed_at`; non-terminal Runはrun-history retention削除対象外
- Execution Log age base: Attempt `completed_at`; running Attempt logは削除しない
- Event age base: Event `created_at`
- Artifact metadata/data age base: Artifact `created_at`
- Managed Artifact dataをmetadataより先にdelete可。`data_deleted_at`記録
- External Artifact dataはCore delete無し
- Output PayloadはRun/Attempt履歴所有物としてrun-history deletion時にdelete

Run history deletionはowned FK rowsが残らないよう依存順に処理する。Component policyが長くてもrun-history expiryが上限。

## 24. 受入条件

1. canonical-json-v1 persistence/digest
2. inline/blob crash consistency
3. retention_policy snapshot
4. non-terminal Run retention safety
5. component retention age bases
6. system-level retention audit survives Run delete
7. Action/Validator snapshot/reuse
8. signed64/string/source identity
9. concurrency/FK semantics
10. deadline indexes
11. Artifact orphan cleanup
12. Retry Input/version
13. uniqueness constraints
14. idempotency original HTTP status metadata
15. future migration reject
