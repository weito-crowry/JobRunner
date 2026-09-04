# 08. Persistence 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/03-runtime-and-scheduling.md`
  - `docs/detailed-design/04-runner-and-ipc.md`
  - `docs/detailed-design/07-external-and-human.md`

## 1. 目的

本書は JobRunner の永続化境界、SQLite schema方針、transaction、migration、index、再起動復元、idempotency保存を定義する。

## 2. 基本原則

1. MVP標準backendはSQLite。
2. JobRunner専用DB fileを既定とする。
3. persistence layerはinterface化し、将来PostgreSQLへ交換可能にする。
4. state transitionはtransaction内で確定する。
5. append-onlyで残すべき履歴は上書きしない。
6. Execution Log本文やArtifact実体はDBに大容量保存しない。
7. migrationはCoreが管理する番号付きSQL方式とする。

## 3. SQLite設定

起動時に少なくとも以下を有効化する。

```text
PRAGMA journal_mode=WAL
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=<configurable>
```

connectionは短命transactionを基本とし、長時間lockを保持しない。

## 4. ID

外部公開IDは衝突しない文字列IDを基本とする。

候補:

```text
UUIDv7 / ULID相当
```

DB内部でINTEGER surrogate keyを併用してもよいが、Service APIでは文字列IDをcanonicalとする。

## 5. 時刻

DBにはUTC時刻を保存する。

canonical表現:

```text
RFC3339 / ISO-8601 UTC
```

またはbackend内部でepoch integerを使用してもよいが、repository層から上はtimezone-aware datetimeとする。

## 6. 主テーブル

MVPで少なくとも以下を持つ。

```text
workflow_runs
job_runs
job_attempts
job_steps
workflow_state
workflow_state_history
artifacts
artifact_aliases optional
events
execution_logs
runners
runner_restarts
external_tasks
external_leases
human_reviews
idempotency_records
workflow_concurrency optional
```

## 7. workflow_runs

主要column:

```text
id PK
workflow_id
workflow_version
status
conclusion nullable
priority
definition_yaml
definition_hash
input_json
source_identity nullable
actor_context_json nullable
access_scope_json nullable
parent_workflow_run_id nullable
parent_job_run_id nullable
parent_attempt_id nullable
root_workflow_run_id
call_depth
pause_requested boolean
cancel_requested boolean
created_at
started_at nullable
completed_at nullable
updated_at
```

必要index:

- `(status, priority, created_at)`
- `(workflow_id, created_at)`
- `parent_workflow_run_id`
- `root_workflow_run_id`

## 8. job_runs

主要column:

```text
id PK
workflow_run_id FK
job_key
job_template_key nullable
dynamic_key nullable
executor
action_id nullable
action_version nullable
runs_on nullable
status
conclusion nullable
priority
continue_on_error boolean
ready_at nullable
queued_at nullable
started_at nullable
completed_at nullable
current_attempt_no
input_json nullable
output_json nullable
failure_json nullable
order_rank nullable
created_at
updated_at
```

unique候補:

```text
(workflow_run_id, job_key)
```

## 9. job_attempts

主要column:

```text
id PK
job_run_id FK
attempt_no
status
conclusion nullable
runner_id nullable
runner_instance_id nullable
input_json
output_json nullable
failure_json nullable
started_at
completed_at nullable
created_at
```

unique:

```text
(job_run_id, attempt_no)
```

Attemptは上書き再利用しない。

## 10. job_steps

主要column:

```text
id PK
attempt_id FK
sequence
name
status
conclusion nullable
started_at
completed_at nullable
metadata_json nullable
```

unique:

```text
(attempt_id, sequence)
```

## 11. workflow_state

current値。

```text
workflow_run_id
name
revision
value_json
updated_at
updated_by_job_run_id nullable
updated_by_attempt_id nullable
updated_by_step_id nullable
```

PKまたはunique:

```text
(workflow_run_id, name)
```

## 12. workflow_state_history

append-only。

```text
id
workflow_run_id
name
revision
old_value_json nullable
new_value_json nullable
job_run_id nullable
attempt_id nullable
step_id nullable
created_at
```

revisionはworkflow_run + name単位で単調増加。

## 13. artifacts

```text
id PK
workflow_run_id
job_run_id
attempt_id
name
uri
media_type nullable
size_bytes nullable
digest nullable
metadata_json nullable
created_at
deleted_at nullable
```

Artifact実体は親側保存。

current aliasが必要な場合は `artifact_aliases` で `workflow_run_id + job_key + name -> artifact_id` を管理してもよい。

## 14. events

append-only。

```text
id PK
workflow_run_id nullable
job_run_id nullable
attempt_id nullable
runner_id nullable
event_type
actor_type nullable
actor_id nullable
source nullable
payload_json
created_at
```

index:

- `(workflow_run_id, created_at)`
- `(job_run_id, created_at)`
- `(event_type, created_at)`

## 15. execution_logs

DBはmetadataのみ。

```text
id
attempt_id
path
size_bytes
first_written_at nullable
last_written_at nullable
created_at
```

Log本文はfilesystem。

## 16. runners

```text
runner_id PK
runner_instance_id
runtime_instance_id
pool_name
status
pid nullable
current_job_run_id nullable
current_attempt_id nullable
started_at
last_heartbeat_at
stopped_at nullable
metadata_json nullable
```

current instance以外の履歴を別tableへ残してもよい。

## 17. runner_restarts

```text
id
runner_id
old_instance_id nullable
new_instance_id nullable
reason
attempt_no
created_at
```

Crash loop抑止判定に利用可能。

## 18. external_tasks / leases

`external_tasks`:

```text
id
job_run_id
attempt_id
status
input_json
created_at
completed_at nullable
```

`external_leases`:

```text
id
external_task_id
lease_id unique
claimed_by
status
claimed_at
expires_at
released_at nullable
```

current valid leaseを一意にするconstraint/indexを持つ。

## 19. human_reviews

```text
id
job_run_id
attempt_id
outcome
comment nullable
actor_context_json nullable
created_at
```

同一Attemptでterminal reviewは1件だけ成功する。

## 20. idempotency_records

```text
scope
operation
request_key
request_hash
result_json
status
created_at
expires_at nullable
```

unique:

```text
(scope, operation, request_key)
```

同じkeyで内容が異なるrequestはconflict。

## 21. Definition snapshot

`workflow_runs.definition_yaml` はWorkflow Run開始時の実使用YAML全文を保存する。

`definition_hash` はcanonical normalization後のDefinition識別用hash。

hashだけ保存してYAML本文を省略することは禁止する。

## 22. JSON canonicalization

永続化するInput / Output / metadataはJSON-compatibleとする。

hash比較対象はcanonical JSONへ変換する。

- object key sort
- UTF-8
- NaN / Infinity禁止
- datetime等はschema上のstringへ正規化

## 23. Transaction境界

最低限以下はatomicにする。

- Workflow Run作成 + initial Job作成
- Job claim + Attempt開始 + Runner ownership
- Dynamic Job expansion全件insert
- task_claim + lease作成
- task_submit + Job完了
- review_submit + Job完了
- state current更新 + history追加
- Retry Attempt作成
- Cancel state transition
- idempotency result確定

## 24. Optimistic / conflict処理

SQLiteではtransaction + conditional UPDATEを基本とする。

例:

```sql
UPDATE job_runs
SET status='running'
WHERE id=? AND status='queued';
```

更新件数0ならclaim conflict。

## 25. Migration

`migrations/` に連番SQLを置く。

例:

```text
001_initial.sql
002_add_concurrency.sql
```

別tableで適用済versionを管理。

migrationは順番に1回だけ適用。

途中失敗時はtransaction rollback可能な範囲でrollbackし、起動をfail-closedにする。

## 26. Backup / portability

MVPで高度なbackup managerは持たない。

SQLite file + run directoryを親システム側でbackupできる構造にする。

DB内pathは可能な限りdata rootからの相対pathを保存する。

## 27. Retention

Retention削除はFK整合を壊さない順序で実行する。

削除前にretention eventを記録する。

Artifact実体の削除責任は親側。

## 28. Repository interface

Core serviceが直接SQLを散在させず、領域ごとのrepository/serviceへ集約する。

例:

```text
WorkflowRunRepository
JobRepository
RunnerRepository
ArtifactRepository
EventRepository
IdempotencyRepository
```

将来backend交換時にRuntime logicを書き換えない。

## 29. Recovery query

起動時に少なくとも以下を効率的に取得できるindexを用意する。

- non-terminal Workflow Runs
- running Jobs
- current Runner instances
- unexpired External leases
- waiting_review Jobs
- queued Jobs

## 30. 受入条件

1. migration fresh install
2. migration incremental
3. migration失敗時fail-closed
4. concurrent Job claim一意性
5. state current+history atomicity
6. dynamic expansion rollback
7. external lease一意性
8. idempotency duplicate replay
9. idempotency key request mismatch conflict
10. restart後non-terminal Run復元
11. FK integrity
12. WAL concurrent read/write
13. retention整合
14. definition snapshot保存
