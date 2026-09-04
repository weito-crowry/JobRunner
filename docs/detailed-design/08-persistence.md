# 08. Persistence 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、transaction、unique/index、Migration、Recovery、Idempotencyの正規契約を定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state / history / inline JSON
├─ Payload metadata
└─ Artifact metadata

PayloadStore
└─ large JSON Output blob

ArtifactStore
└─ Actionが明示登録したmanaged artifact実体
```

MVP標準:

- SQLite: Python stdlib `sqlite3`
- PayloadStore: `LocalHybridPayloadStore`
  - <= inline threshold: SQLite
  - > threshold: durable filesystem JSON blob
- ArtifactStore: `LocalArtifactStore`
  - managed artifactをdurable filesystemへcopy

Storage interfaceを抽象化し将来backend交換可能にする。

## 3. SQLite設定

Connection:

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000  # default configurable
```

DB:

```text
PRAGMA journal_mode=WAL
```

競合writeは必要に応じ `BEGIN IMMEDIATE`。長transaction禁止。

## 4. ID / time / JSON

外部IDはtype prefix + UUID4 hex。時刻はUTC RFC3339。

Canonical JSON: UTF-8、NaN/Infinity禁止、object key deterministic serialization。

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

Payload blobはtableを増やさずWorkflow Run / Attempt rowのstorage metadataで管理する。

## 6. Output payload columns

Workflow OutputとAttempt Outputは同じshape。

```text
output_storage_kind  NULL | inline | blob
output_json          NULL | TEXT
output_blob_key      NULL | TEXT
output_size_bytes    NULL | INTEGER >=0
output_digest        NULL | TEXT
```

Constraint:

- no output: all null
- inline: `output_json NOT NULL`, blob_key null
- blob: `output_json NULL`, blob_key NOT NULL
- size/digestはoutputありなら必須

Blob keyはdata root相対のCore生成key。外部pathを受け取らない。

## 7. PayloadStore write/read

Write:

1. JSON/schema/success_if/SecretGuard完了
2. canonical JSON bytes作成
3. digest SHA-256
4. threshold比較
5. inlineまたはtemp blobへwrite
6. DB state transition transactionでmetadata確定
7. blobの場合DB commit後もdurable fileが存在することを保証

### 7.1 crash consistency

Blob writeは:

```text
<key>.tmp -> fsync/close -> atomic rename -> DB metadata commit
```

を基本とする。

DB commit前crashでorphan blobが残ることは許容し、maintenanceで削除。

DBがblobを参照しているのにfileが無い状態は通常operationでは作らない。

Read時にfile existence/size/digestを検証し、不整合は `payload_missing` / `payload_digest_mismatch`。

### 7.2 Transparent read

Repository/Serviceはstorage kindを隠蔽し、callerへ元のJSON valueを返す。

## 8. `workflow_runs`

主要:

```text
id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
pause_requested/cancel_requested

definition_yaml/definition_json/definition_hash
input_json/action_versions_json
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
failure_json/source_identity

concurrency_group/max/on_limit
actor_context_json/access_scope_json
parent_workflow_run_id/parent_job_run_id/parent_attempt_id/root_workflow_run_id/call_depth/reusable_binding_id
created_at/started_at/completed_at/updated_at
```

## 9. `job_runs`

Job logical/current state:

```text
id/workflow_run_id/job_key/template/dynamic parent metadata
executor/action/version/uses/runs_on
status/conclusion/priority/continue_on_error
timeout_seconds/retry_policy_json/retry_not_before
current_attempt_no/current_attempt_id
ready/queue/start/complete timestamps
current_failure_json
```

**Output payload実体/metadataのSource of Truthはsuccessful `job_attempts`。** `job_runs`へOutputを二重保存しない。

`UNIQUE(workflow_run_id, job_key)`。

One running internal:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
```

## 10. `job_attempts`

```text
id/job_run_id/attempt_no
status/conclusion
runner_id/runner_instance_id/runtime_instance_id
input_json
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
reuse_key nullable
failure_json
started_at/completed_at/created_at
```

`UNIQUE(job_run_id, attempt_no)`。

Attempt immutable history。Current Job Outputはcurrent successful Attemptから解決。

## 11. Dynamic / Reusable

`dynamic_expansions`はroot/nested partial unique index。

Full logical `job_key`はTEXT、固定長CHECK無し。

`reusable_bindings`: one binding / parent_job_run_id。

Child Run: one / parent_attempt_id partial unique。

## 12. Workflow State

Current + append-only history。同transaction。

State valueはJSON-compatible。SecretGuard通過必須。

## 13. Artifact metadata

```text
id
workflow_run_id
job_run_id
attempt_id
name
storage_kind      # managed | external
store_key nullable
external_uri nullable
media_type nullable
size_bytes nullable
digest nullable
metadata_json nullable
created_at
data_deleted_at nullable
metadata_deleted_at nullable
```

Constraint:

- managed: `store_key`必須、external_uri null
- external: `external_uri`必須、store_key null

Managed Artifactの実体はArtifactStore。External Artifactの実体は親側。

## 14. ArtifactStore transaction boundary

Managed `put_file`:

1. source pathをAttempt work_dir内へcanonicalize
2. Coreがartifact_id/store_keyを発行
3. ArtifactStoreへtemp copy + digest/size計算
4. atomic finalize
5. Artifact metadata transaction insert + Event

DB insert失敗時のorphan store objectはmaintenance削除。

External `register_reference`はmetadata transactionのみでCoreがURI fetchしない。

## 15. Events / Logs / Runners

Eventsはappend-only structured data。SecretGuard通過。

Execution Log DBはrelative path/size/time metadataのみ。

Runner fencing用にrunner/runtime instance、heartbeat、main_loop_tickを保存。

## 16. External / Human uniqueness

- one External Task / Attempt
- one active Lease / Task partial unique
- one Human Review / Attempt

## 17. Idempotency

```text
scope/operation/request_key PK
request_hash/result_json/status/created_at/expires_at
```

Default TTL 24h。

Scope:

- system namespace
- resource scope
- AccessScope canonical identity
- Actor/client principal

TTL内same hash replay、different hash conflict。

Expiry後rowが残っていても同transactionでexpired rowを置換しsame keyをnew requestとして利用可能。

## 18. Concurrency / Retry transactions

Workflow start/slot releaseは `BEGIN IMMEDIATE` でholder count再確認。

Manual Retry reopen:

- run_attempt++
- completed/failure Runをreopen
- target failed Job retry waiting
- blocked/skipped descendants再評価
- successful descendantは`03`のResult Reuse検証対象

## 19. Migration

`migrations/NNN_name.sql` + `schema_migrations`。順序通り1回、失敗fail-closed。

## 20. Retention / cleanup

Default無期限。

- Output blob: owning Run/Attempt retentionと一緒にPayloadStoreからdelete
- managed Artifact: ArtifactStoreから実体delete + `data_deleted_at`
- external Artifact: Coreは外部実体をdeleteしない
- orphan `.tmp` / unreferenced blob/store objectはmaintenance削除
- idempotency expired record削除可能

## 21. 受入条件

1. inline/spill boundary
2. blob crash consistency/orphan cleanup
3. transparent load
4. missing/digest mismatch
5. arbitrary JSON Output
6. managed ArtifactStore copy
7. external Artifact reference no fetch/delete
8. internal running unique
9. Dynamic/Reusable/External/Human unique
10. state/Event SecretGuard
11. idempotency scope/TTL
12. retention managed data deletion
