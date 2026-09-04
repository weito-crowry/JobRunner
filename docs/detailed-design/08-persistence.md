# 08. Persistence 詳細設計

- Status: Draft v1.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- Canonical JSON: `01-workflow-definition.md` の `jobrunner.canonical-json.v1`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Action/Validator snapshot、Dynamic expansion、Result Reuse、deadline、Retention、Migration、Recovery、Idempotencyを定義する。

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

Run start時effective policyをcanonical JSONでsnapshot。

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

state_get/persistent Input外Artifact materialize -> ineligible。

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

## 15. `dynamic_expansions`

Canonical columns:

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
template_id TEXT NOT NULL
parent_generated_job_run_id TEXT NULL FK
outcome TEXT NOT NULL
source_snapshot_json TEXT NULL
source_digest TEXT NULL
generated_count INTEGER NOT NULL DEFAULT 0
failure_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

`outcome`:

```text
pending|expanded|skipped|failed|cancelled
```

Checks:

```text
generated_count >= 0
pending -> completed_at NULL
expanded|skipped|failed|cancelled -> completed_at NOT NULL
expanded -> source_snapshot_json/source_digest NOT NULL
skipped|cancelled -> generated_count=0
failed -> failure_json NOT NULL
```

Rootでは`parent_generated_job_run_id IS NULL`、NestedではNOT NULL。

SQLiteは`NULL`を含む通常UNIQUEだけではRoot重複を防げないため、以下を必須とする。

```sql
CREATE UNIQUE INDEX uq_dynamic_expansion_root
ON dynamic_expansions(workflow_run_id, template_id)
WHERE parent_generated_job_run_id IS NULL;

CREATE UNIQUE INDEX uq_dynamic_expansion_nested
ON dynamic_expansions(workflow_run_id, template_id, parent_generated_job_run_id)
WHERE parent_generated_job_run_id IS NOT NULL;
```

Expansion commit transactionは:

- expansion outcome/source snapshot
- 全generated `job_runs`
- generated count

をall-or-nothingで確定する。

`if=false` は`skipped`、`foreach=[]`は`expanded/generated_count=0`。この差をRecoveryで維持する。

Nested parent generated Job 0件の直接propagationはgenerated expansion rowを捏造せず、template group resolution metadata/stateから`05`のparent conclusionをidempotentに導出する。

## 16. `reusable_bindings`

最低限:

```text
id TEXT PK
parent_workflow_run_id TEXT NOT NULL FK
parent_job_run_id TEXT NOT NULL FK
workflow_ref_original TEXT NOT NULL
child_workflow_id TEXT NOT NULL
child_workflow_version INTEGER NOT NULL
child_definition_yaml/json/hash
child_action_versions_json TEXT NOT NULL
child_validator_versions_json TEXT NOT NULL
created_at TEXT NOT NULL
UNIQUE(parent_job_run_id)
```

Child Runは`parent_attempt_id`に対して最大1件となるpartial/unique constraintを持つ。

## 17. State

Current + append-only history same transaction。SecretGuard対象。

## 18. Artifact metadata / Store

```text
id/workflow_run_id/job_run_id/attempt_id/name
storage_kind=managed|external
store_key/external_uri
media_type/size/digest/metadata
created_at/data_deleted_at/metadata_deleted_at
```

Managed put=temp copy/Secret scan/digest/atomic finalize/metadata Event。DB failure orphan cleanup。

## 19. Event persistence / retention audit

通常Eventは`workflow_run_id`等をnullable参照として持つ。

System-level retention audit Event:

- `workflow_run_id/job_run_id/attempt_id=NULL`
- payloadへdeleted resource type / opaque ID / workflow_id copy / count / policy key / cutoff
- 対象Run等へのFK無し
- SecretGuard対象

Type=`retention_deleted|retention_orphan_cleaned`。

通常`event-days`対象から除外しMVP無期限保持。Repeated sweepで二重auditしない。

## 20. External / Human uniqueness

Required DB guarantees:

```text
UNIQUE(external_tasks.attempt_id)
UNIQUE(human_reviews.attempt_id)
```

External LeaseはTaskあたりactive最大1をpartial uniqueで保証する。

```sql
CREATE UNIQUE INDEX uq_external_active_lease
ON external_leases(task_id)
WHERE status='active';
```

Task status=`available|leased|completed|cancelled`。
Lease status=`active|expired|released|invalidated`。
Review status=`pending|completed|cancelled`。

## 21. Idempotency

```text
scope/operation/request_key PK
request_hash/result_json/adapter_meta_json/status/created_at/expires_at
```

Default TTL24h。Scope=namespace+resource+AccessScope+Actor/client principal。

TTL内replay/conflict、expired replace可。Result/adapter metadata SecretGuard対象。

Side effect + idempotency result same transaction。

## 22. Concurrency

Start/slot release `BEGIN IMMEDIATE` holder recount。Group BINARY case-sensitive。

## 23. Migration

`migrations/NNN_name.sql` + `schema_migrations`。Ordered once、fail-closed、unknown future version reject。

## 24. Retention deletion rules

Policy source=`workflow_runs.retention_policy_json`。

- Run age base: `completed_at`; non-terminal Runはrun-history retention削除対象外
- Execution Log age base: Attempt `completed_at`; running Attempt logは削除しない
- Event age base: Event `created_at`; system-level retention audit Eventは除外
- Artifact metadata/data age base: Artifact `created_at`; owner Run non-terminal中は削除しない
- Managed Artifact dataをmetadataより先にdelete可。`data_deleted_at`記録
- External Artifact dataはCore delete無し
- Output PayloadはRun/Attempt履歴所有物としてrun-history deletion時にdelete

Run history deletionはowned FK rowsが残らないよう依存順に処理する。Component policyが長くてもrun-history expiryが上限。

Artifact metadata削除時にManaged dataが残る状態を作らない。必要ならdata deleteを先行する。

Orphan temp/blob/store objectはconsistency cleanupとして削除可能で、system-level `retention_orphan_cleaned` audit Eventを残す。

## 25. 受入条件

1. canonical-json-v1 persistence/digest
2. inline/blob crash consistency
3. dynamic root/nested partial unique
4. dynamic skip vs empty outcome persistence
5. dynamic atomic expansion/recovery
6. reusable binding unique/version snapshot
7. external Task/active Lease/Review uniqueness
8. retention_policy snapshot
9. non-terminal Run retention safety
10. component retention age bases
11. system-level retention audit survives Run delete/event sweep
12. Action/Validator snapshot/reuse
13. signed64/string/source identity
14. concurrency/FK semantics
15. deadline indexes
16. Artifact orphan cleanup audit
17. Retry Input/version
18. idempotency original HTTP status metadata
19. future migration reject
