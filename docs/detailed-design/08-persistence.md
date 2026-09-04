# 08. Persistence 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Result Reuse metadata、transaction、Migration、Recovery、Idempotencyを定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ Payload metadata
├─ Artifact metadata
└─ Reuse metadata

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

## 4. ID / time / JSON

IDはtype prefix + UUID4 hex。UTC RFC3339。

Canonical JSONはUTF-8、NaN/Infinity禁止、deterministic object serialization。

## 5. Tables

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

## 6. Output payload columns

Workflow Run / Job AttemptのOutput:

```text
output_storage_kind  NULL | inline | blob
output_json          NULL | TEXT
output_blob_key      NULL | TEXT
output_size_bytes    NULL | INTEGER >=0
output_digest        NULL | TEXT
```

Constraint:

- none: all null
- inline: output_json set / blob_key null
- blob: output_json null / blob_key set
- outputありならsize/digest必須

## 7. PayloadStore crash consistency

Write:

1. result validation + SecretGuard
2. canonical JSON bytes + SHA-256
3. threshold判定
4. blobなら `<key>.tmp` write/close -> atomic rename
5. DB terminal transactionでmetadata commit

DB commit前crashのorphan blobはmaintenance cleanup。

Readでexistence/size/digest検証。不整合:

```text
payload_missing
payload_digest_mismatch
```

Repository/Serviceはinline/blob差を隠し元JSON valueを返す。

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

```text
id
workflow_run_id
job_key/job_template_key/dynamic_key/parent_job_run_id/dynamic_expansion_id
item_snapshot_json/iteration_context_json/source_order/order_rank
executor/action_id/action_version/uses_ref/reusable_binding_id/runs_on
status/conclusion
terminal_reason nullable
priority/continue_on_error
timeout_seconds/retry_policy_json/retry_not_before
current_attempt_no/current_attempt_id
reuse_check_pending INTEGER NOT NULL DEFAULT 0 CHECK IN(0,1)
current_failure_json
ready_at/queued_at/started_at/completed_at/created_at/updated_at
```

`terminal_reason` example:

```text
condition_false
dependency_skipped
dependency_failed
dynamic_empty
cancelled
execution_terminal
```

`UNIQUE(workflow_run_id, job_key)`。

One running internal:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
```

Output Source of Truthはsuccessful `job_attempts`。

## 10. `job_attempts`

```text
id/job_run_id/attempt_no
status/conclusion
runner_id/runner_instance_id/runtime_instance_id
input_json
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
reuse_eligible INTEGER NULL CHECK IN(0,1)
reuse_context_json TEXT NULL
reuse_key TEXT NULL
failure_json
started_at/completed_at/created_at
```

Successful Attemptでは `reuse_eligible/reuse_context/reuse_key` を確定する。

`reuse_context_json`はnon-secret:

```text
workflow_run_id
job_key
definition_hash
persistent_input_digest
direct_upstream_artifacts
executor_identity
ineligibility_reason optional
```

`UNIQUE(job_run_id, attempt_no)`。

## 11. Reuse key transaction

Execution activation時にbase reuse contextを作る。Runtime Handle利用によりeligibilityが変わり得るため、最終 `reuse_eligible/key` はsuccessful terminal transition時に確定する。

- `state_get` 使用 -> reuse_eligible=false
- undeclared/dynamic Artifact materialize -> false
- otherwise canonical reuse context SHA-256

Output Payload commitとreuse metadataとJob success transitionは同じlogical terminal operationで確定する。

## 12. Manual Retry reopen / reuse markers

Completed/failure Runのmanual retry transaction:

1. `run_attempt += 1`
2. Run reopen
3. target failed Job -> retry waiting
4. target dependency closureのblocked/skipped descendants -> non-terminal activation待ちへ戻す
5. successful descendants -> `reuse_check_pending=1`
6. Event/idempotency

Successful JobのAttempt/Output/Artifactを削除しない。

Reuse check結果:

- match -> `reuse_check_pending=0`, success維持、`job_result_reused`
- mismatch/ineligible/payload missing -> Run failure `successful_job_result_not_reusable`

MVPでは同Job Runへnew Inputのre-executionを自動作成しない。

## 13. Dynamic / Reusable uniqueness

`dynamic_expansions`: root/nested partial unique。Full `job_key` TEXT、固定長CHECK無し。

`reusable_bindings`: one / parent_job_run_id。

Child Run: one / parent_attempt_id partial unique。

## 14. State

Current + append-only historyを同transaction。State valueはSecretGuard対象。

## 15. Artifact metadata / Store

```text
id/workflow_run_id/job_run_id/attempt_id/name
storage_kind = managed|external
store_key nullable
external_uri nullable
media_type/size/digest/metadata
created_at/data_deleted_at/metadata_deleted_at
```

ManagedはArtifactStore、Externalはparent data。

Managed putはwork_dir内source -> temp copy/Secret scan/digest -> atomic finalize -> metadata/Event。

## 16. Events / Logs / Runners

Eventはappend-only + SecretGuard。

Execution Log DBはrelative path metadataのみ。

Runner fencingにruntime/runner instance + heartbeat/main_loop_tick。

## 17. External / Human uniqueness

- one Task / Attempt
- one active Lease / Task
- one Review / Attempt

## 18. Idempotency

```text
scope/operation/request_key PK
request_hash/result_json/status/created_at/expires_at
```

Default TTL24h。

Scope includes system namespace + resource + AccessScope + Actor/client principal。

TTL内replay/conflict。Expiry後row残存でも同transactionで置換してnew requestとして利用可能。

## 19. Concurrency

Workflow start/slot releaseは`BEGIN IMMEDIATE`でholder count再確認。

## 20. Migration

`migrations/NNN_name.sql` + `schema_migrations`。順序1回、fail-closed。

## 21. Retention

Default無期限。

- Output blob: owner Run/Attempt retentionでdelete
- managed Artifact: ArtifactStore delete
- external Artifact: external data deleteしない
- orphan temp/blob/store object cleanup
- expired idempotency cleanup

Successful result reuse対象Payload/managed Artifactがretentionで消えた場合、その結果は後続reuse checkで不適格になる。

## 22. 受入条件

1. inline/blob constraints/crash consistency
2. arbitrary JSON transparent read
3. managed/external Artifact
4. reuse_context/key persistence
5. state_get reuse ineligible
6. Manual Retry reuse_check_pending
7. reuse match/mismatch transaction
8. one-running/Dynamic/Reusable/External/Human unique
9. idempotency/concurrency
10. retention causes reuse failure not silent reuse
