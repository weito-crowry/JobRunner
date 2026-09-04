# 08. Persistence 詳細設計

- Status: Draft v0.8
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Action/Validator snapshot、Result Reuse、deadline、Migration、Recovery、Idempotencyを定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ Payload metadata
├─ Artifact metadata
├─ Action/Validator identity
└─ Reuse/idempotency metadata

PayloadStore
└─ large JSON Output blob

ArtifactStore
└─ explicitly managed artifact data
```

MVP standard:

- `sqlite3`
- `LocalHybridPayloadStore`: <= threshold SQLite / > threshold filesystem
- `LocalArtifactStore`: durable filesystem

## 3. SQLite設定

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000
PRAGMA journal_mode=WAL
```

競合writeは必要に応じ`BEGIN IMMEDIATE`。長transaction禁止。

Text identity比較はSQLite `BINARY` semanticsを基本とし、Concurrency groupはcase-sensitive完全一致。

## 4. ID / time / JSON / numeric

IDはtype prefix + UUID4 hex。UTC RFC3339。

Canonical JSONはUTF-8、NaN/Infinity禁止、deterministic object serialization。

SQLite INTEGERへ保存する外部入力整数はsigned 64-bit範囲。Priority/version/count等はDB到達前validation。

Action/Validator ID+version、Runner Pool、Concurrency group等の必須identityはnon-empty TEXT。

## 5. Foreign Key policy

Implicit cascade delete無し。

- FK原則 `ON DELETE NO ACTION`
- Retention Serviceが依存順に明示削除
- FK無効化で強行しない

少なくとも leaf data -> attempt-owned -> job-owned -> child runs -> root run の順。

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

Maintenance Loop専用tableは作らず `job_runs.retry_not_before` と `external_leases.expires_at` をdeadline Source of Truthにする。

## 7. Output payload columns

Workflow Run / Job Attempt:

```text
output_storage_kind  NULL | inline | blob
output_json          NULL | TEXT
output_blob_key      NULL | TEXT
output_size_bytes    NULL | INTEGER >=0
output_digest        NULL | TEXT
```

- none: all null
- inline: output_json set / blob_key null
- blob: output_json null / blob_key set
- outputありならsize/digest必須

## 8. PayloadStore crash consistency

1. result validation + SecretGuard
2. canonical JSON + SHA-256
3. threshold
4. blob temp write -> atomic rename
5. DB terminal transaction metadata

DB commit前crashのorphan blobはmaintenance cleanup。

Readでexistence/size/digest verify。不整合 `payload_missing|payload_digest_mismatch`。

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
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
failure_json/source_identity

concurrency_group TEXT COLLATE BINARY NULL
concurrency_max_runs/concurrency_on_limit
actor_context_json/access_scope_json
parent_workflow_run_id/parent_job_run_id/parent_attempt_id/root_workflow_run_id/call_depth/reusable_binding_id
created_at/started_at/completed_at/updated_at
```

Checks:

```text
status = queued|running|paused|completed
conclusion = NULL|success|failure|cancelled
completed <=> conclusion non-NULL
concurrency_group NULL or length>0
concurrency_max_runs NULL or >=1
concurrency_on_limit NULL|queue|reject
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

Checks:

```text
executor = internal|external_llm|human|reusable
status = queued|running|waiting_external|waiting_review|waiting_child|completed
conclusion = NULL|success|failure|cancelled|skipped|blocked
internal -> action_id/action_version non-empty, runs_on non-empty
external/human/reusable -> action_id/action_version/runs_on NULL
validator pair either both NULL or both non-empty
human/reusable -> validator pair NULL
internal timeout_seconds NULL or finite >0
```

`UNIQUE(workflow_run_id, job_key)`。

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
reuse_context_json TEXT NULL
reuse_key TEXT NULL
failure_json
started_at/completed_at/created_at
UNIQUE(job_run_id, attempt_no)
```

Attempt status:

```text
running|waiting_external|waiting_review|waiting_child|completed
```

Attempt開始前failureはJob Run failureで表現しInput snapshot無し。

## 12. Reuse context

Successful terminal時に:

```text
workflow_run_id
job_key
definition_hash
persistent_input_digest
direct_upstream_artifacts
executor_identity
validator_identity
ineligibility_reason optional
```

を確定。

- state_get -> ineligible
- undeclared/dynamic Artifact materialize -> ineligible
- otherwise canonical context SHA-256 `reuse_key`

Output commit + reuse metadata + Job successを同logical terminal operationで確定。

## 13. Manual Retry reopen

1. latest failed Attempt + input_json存在
2. 無ければ `retry_input_unavailable`, no state change
3. current Action/Validator/Reusable binding version availability確認
4. run_attempt++
5. Run reopen
6. target retry waiting
7. blocked/skipped descendants reset
8. successful descendants reuse_check_pending=1
9. Event/idempotency

Reuse mismatch/version unavailable/ineligible/payload missingは `successful_job_result_not_reusable` または明示version mismatch。

## 14. Deadline indexes / Maintenance

```sql
CREATE INDEX ix_job_retry_due
ON job_runs(retry_not_before)
WHERE retry_not_before IS NOT NULL AND status='queued';

CREATE INDEX ix_external_lease_due
ON external_leases(expires_at)
WHERE status='active';
```

Deadline処理はconditional transaction。二重expiry/retry wakeを副作用化しない。

## 15. Dynamic / Reusable uniqueness

Dynamic expansion root/nested partial unique。Full `job_key` TEXT固定長CHECK無し。

Reusable binding one / parent_job_run_id。BindingはChild Action+Validator version snapshotを持つ。

Child Run one / parent_attempt_id partial unique。

## 16. State

Current + append-only history同transaction。State valueはSecretGuard対象。

## 17. Artifact metadata / Store

```text
id/workflow_run_id/job_run_id/attempt_id/name
storage_kind = managed|external
store_key/external_uri
media_type/size/digest/metadata
created_at/data_deleted_at/metadata_deleted_at
```

Managed putはwork_dir内source -> temp copy/Secret scan/digest -> atomic finalize -> metadata/Event。

Store finalize後DB failureはorphan cleanup対象。

## 18. Events / Logs / Runners

Event append-only + SecretGuard。Execution Log DBはrelative path metadata。

Runner fencingにruntime/runner instance + heartbeat/main_loop_tick。

## 19. External / Human uniqueness

- one Task / Attempt
- one active Lease / Task
- one Review / Attempt

## 20. Idempotency records

Canonical schema:

```text
scope TEXT NOT NULL
operation TEXT NOT NULL
request_key TEXT NOT NULL
request_hash TEXT NOT NULL
result_json TEXT NOT NULL
adapter_meta_json TEXT NULL
status TEXT NOT NULL
created_at TEXT NOT NULL
expires_at TEXT NOT NULL
PRIMARY KEY(scope, operation, request_key)
```

Default TTL=24h。

Scope includes namespace + resource + AccessScope + Actor/client principal。

TTL内:

- same hash -> first Service result replay
- different hash -> idempotency_conflict

TTL後:

- expired rowが残っていても同transactionでreplaceしてnew requestを受理可能

`adapter_meta_json` はService意味を変えないtransport replay補助metadata。HTTP Adapterは初回successful status code (`200|201`等)を保存でき、replayでoriginal statusを復元する。MCP/Python固有transport object本体をDBへ保存しない。

`result_json` / `adapter_meta_json` はSecretGuard対象。

Side effect + Service result + idempotency rowは同transaction。Filesystem blob prepareを伴う場合はPayloadStore crash consistencyに従う。

## 21. Concurrency

Start/slot releaseは`BEGIN IMMEDIATE`でholder count再確認。Group non-empty TEXT `COLLATE BINARY`。

## 22. Migration

`migrations/NNN_name.sql` + `schema_migrations`。順序1回、fail-closed。Unknown future schema versionは起動拒否。

## 23. Retention

Default無期限。

- Output blob owner retention delete
- managed ArtifactStore delete
- external Artifact実体deleteしない
- orphan temp/blob/store cleanup
- expired idempotency cleanup
- relational rows依存順削除

Reuse対象Payload/Artifact欠落はsilent reuseしない。

## 24. 受入条件

1. inline/blob constraints/crash consistency
2. Action/Validator snapshot columns
3. validator identity reuse key
4. signed64/string/source_identity boundaries
5. concurrency BINARY
6. FK NO ACTION / retention ordering
7. deadline indexes / idempotent due processing
8. managed/external Artifact
9. Manual Retry Input/version existence
10. one-running/Dynamic/Reusable/External/Human unique
11. idempotency adapter_meta original HTTP status replay
12. expired idempotency replace
13. migration future version reject
