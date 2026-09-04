# 08. Persistence 詳細設計

- Status: Draft v0.6
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

Text identity比較で明示しない限りSQLite `BINARY` semanticsを使う。Concurrency groupは `COLLATE BINARY` のcase-sensitive完全一致。

## 4. ID / time / JSON / numeric

IDはtype prefix + UUID4 hex。UTC RFC3339。

Canonical JSONはUTF-8、NaN/Infinity禁止、deterministic object serialization。

SQLite INTEGERへ保存する外部入力整数はsigned 64-bit範囲。Priority/version/count等はService/model validationで範囲外をDB到達前にrejectする。

Action ID/version、Runner Pool、Concurrency group等の必須string identityはnon-empty TEXT。

## 5. Foreign Key policy

MVPの参照整合性は **implicit cascade deleteを使わない**。

- FKは原則 `ON DELETE NO ACTION`
- 親Run削除で子履歴を暗黙消去しない
- Retention Serviceが依存row/Blob/Artifactを明示順序で削除する
- Root Workflow RunはChild/Job/Attempt/Artifact/Event等の保持対象が残る間は削除不可
- Child Workflow RunはParent relationを保持したまま単独で親より先に削除しない

Retentionは少なくとも leaf data -> attempt-owned rows -> job-owned rows -> child runs -> root run の順序で処理する。失敗したらtransaction/maintenance単位でfail-closedし、FK無効化で強行しない。

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

## 7. Output payload columns

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

## 8. PayloadStore crash consistency

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

## 9. `workflow_runs`

主要:

```text
id TEXT PK
workflow_id TEXT NOT NULL
workflow_version INTEGER NOT NULL CHECK(workflow_version >= 1)
status TEXT NOT NULL
conclusion TEXT NULL
priority INTEGER NOT NULL
run_attempt INTEGER NOT NULL CHECK(run_attempt >= 1)
wait_reason TEXT NULL
pause_requested INTEGER NOT NULL CHECK IN(0,1)
cancel_requested INTEGER NOT NULL CHECK IN(0,1)

definition_yaml TEXT NOT NULL
definition_json TEXT NOT NULL
definition_hash TEXT NOT NULL
input_json TEXT NOT NULL
action_versions_json TEXT NOT NULL
output_storage_kind/output_json/output_blob_key/output_size_bytes/output_digest
failure_json/source_identity

concurrency_group TEXT COLLATE BINARY NULL
concurrency_max_runs INTEGER NULL
concurrency_on_limit TEXT NULL

actor_context_json/access_scope_json
parent_workflow_run_id/parent_job_run_id/parent_attempt_id/root_workflow_run_id/call_depth/reusable_binding_id
created_at/started_at/completed_at/updated_at
```

Checks:

```text
status = queued|running|paused|completed
conclusion = NULL|success|failure|cancelled
completed <=> conclusion non-NULL
concurrency_group is NULL OR length(group)>0
concurrency_max_runs is NULL OR >=1
concurrency_on_limit is NULL|queue|reject
```

## 10. `job_runs`

```text
id TEXT PK
workflow_run_id TEXT NOT NULL FK
job_key TEXT NOT NULL
job_template_key/dynamic_key/parent_job_run_id/dynamic_expansion_id
item_snapshot_json/iteration_context_json/source_order/order_rank
executor TEXT NOT NULL
action_id TEXT NULL
action_version TEXT NULL
uses_ref/reusable_binding_id/runs_on
status TEXT NOT NULL
conclusion TEXT NULL
terminal_reason NULL
priority INTEGER NOT NULL
continue_on_error INTEGER NULL CHECK IN(0,1)
timeout_seconds REAL NULL
retry_policy_json/retry_not_before
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
non-internal -> action_id/action_version/runs_on NULL
internal timeout_seconds NULL or finite >0
```

`UNIQUE(workflow_run_id, job_key)`。

One running internal:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';
```

Output Source of Truthはsuccessful `job_attempts`。

## 11. `job_attempts`

```text
id TEXT PK
job_run_id TEXT NOT NULL FK
attempt_no INTEGER NOT NULL CHECK(attempt_no >=1)
status TEXT NOT NULL
conclusion TEXT NULL
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

Attemptは実execution開始時のみ作る。Attempt開始前failureは`job_runs.current_failure_json`等で表現し、Retry Input snapshotは存在しない。

Successful Attemptでは `reuse_eligible/reuse_context/reuse_key` を確定する。

## 12. Reuse key transaction

Execution activation時にbase reuse contextを作る。Runtime Handle利用によりeligibilityが変わり得るため、最終 `reuse_eligible/key` はsuccessful terminal transition時に確定する。

- `state_get` 使用 -> reuse_eligible=false
- undeclared/dynamic Artifact materialize -> false
- otherwise canonical reuse context SHA-256

Output Payload commitとreuse metadataとJob success transitionは同じlogical terminal operationで確定する。

## 13. Manual Retry reopen / reuse markers

Completed/failure Runのmanual retry transaction:

1. target latest failed Attempt + persistent `input_json` 存在を確認
2. 無ければ `retry_input_unavailable` でtransaction変更なし
3. `run_attempt += 1`
4. Run reopen
5. target failed Job -> retry waiting
6. blocked/skipped descendants -> non-terminal activation待ち
7. successful descendants -> `reuse_check_pending=1`
8. Event/idempotency

Successful JobのAttempt/Output/Artifactを削除しない。

Reuse check結果:

- match -> `reuse_check_pending=0`, success維持、`job_result_reused`
- mismatch/ineligible/payload missing -> Run failure `successful_job_result_not_reusable`

MVPでは同Job Runへnew Inputのre-executionを自動作成しない。

## 14. Dynamic / Reusable uniqueness

`dynamic_expansions`: root/nested partial unique。Full `job_key` TEXT、固定長CHECK無し。

`reusable_bindings`: one / parent_job_run_id。

Child Run: one / parent_attempt_id partial unique。

## 15. State

Current + append-only historyを同transaction。State valueはSecretGuard対象。

## 16. Artifact metadata / Store

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

Store finalize後DB commit失敗時はorphan objectとしてmaintenance cleanup対象。Metadataが無いobjectをcurrentとして露出しない。

## 17. Events / Logs / Runners

Eventはappend-only + SecretGuard。

Execution Log DBはrelative path metadataのみ。

Runner fencingにruntime/runner instance + heartbeat/main_loop_tick。

## 18. External / Human uniqueness

- one Task / Attempt
- one active Lease / Task
- one Review / Attempt

## 19. Idempotency

```text
scope/operation/request_key PK
request_hash/result_json/status/created_at/expires_at
```

Default TTL24h。

Scope includes system namespace + resource + AccessScope + Actor/client principal。

TTL内replay/conflict。Expiry後row残存でも同transactionで置換してnew requestとして利用可能。

## 20. Concurrency

Workflow start/slot releaseは`BEGIN IMMEDIATE`でholder count再確認。

Groupはnon-empty TEXT `COLLATE BINARY`。同一group判定はcase-sensitive。暗黙normalization無し。

## 21. Migration

`migrations/NNN_name.sql` + `schema_migrations`。順序1回、fail-closed。

Migrationは適用済みversionを飛ばさず、unknown future schema versionを開いた場合は起動拒否する。

## 22. Retention

Default無期限。

- Output blob: owner Run/Attempt retentionでdelete
- managed Artifact: ArtifactStore delete
- external Artifact: external data deleteしない
- orphan temp/blob/store object cleanup
- expired idempotency cleanup
- relational rowsはFK依存順に明示削除しcascadeに依存しない

Successful result reuse対象Payload/managed Artifactがretentionで消えた場合、その結果は後続reuse checkで不適格になる。

## 23. 受入条件

1. inline/blob constraints/crash consistency
2. arbitrary JSON transparent read
3. signed64/numeric/string identity boundaries
4. concurrency BINARY case-sensitive group
5. FK NO ACTION / retention delete ordering
6. managed/external Artifact + orphan store cleanup
7. reuse_context/key persistence
8. state_get reuse ineligible
9. Manual Retry Input snapshot existence
10. reuse match/mismatch transaction
11. one-running/Dynamic/Reusable/External/Human unique
12. idempotency/concurrency
13. unknown future migration version reject
14. retention causes reuse failure not silent reuse
