# 11. Service API / MCP 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03-runtime-and-scheduling.md`, `06-reusable-workflows.md`, `07-external-and-human.md`, `10-retry-recovery-cancel.md`, `12-security-and-secrets.md`

## 1. 目的

Python/Web/MCPが共有するService operation、Definition/Runの区別、request/response、Authorization、Idempotency、namespace、Errorを定義する。

## 2. 基本原則

1. Python/Web/MCPは同じService layerを呼ぶ。
2. AdapterはDB/Repositoryを直接更新しない。
3. **read/writeを含む全Service operationがAuthorizationProviderを通る。**
4. State-changing operationはoptional `request_id` idempotencyを持つ。
5. Side effect/Event/idempotency resultは可能な限り同一transaction。
6. MCP public toolは親システムnamespace必須。
7. DefinitionとWorkflow Runをoperation名で明確に分ける。
8. 大きいExecution Log本文をRun infoへ埋め込まない。

## 3. Service構成

```text
WorkflowDefinitionService
WorkflowRunService
JobService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
RunnerService
```

外部Facadeは統合しても内部責務は分離。

## 4. ActorContext / request metadata

全operation:

```text
actor_context
```

state-changing:

```text
request_id optional
```

`request_id`はidempotency key。Authorizationはidempotency replayより先に実施する。

## 5. Workflow Definition read operations

### `wf_definition_list`

利用可能Workflow設計図一覧。

filter:

```text
workflow_id/name optional
limit/cursor
```

返却:

```text
workflow_id
name
version
description
definition_hash
input schema summary
```

### `wf_definition_info`

request:

```text
workflow_ref
include_source_yaml=false
```

返却:

```text
canonical workflow_id
name/version/description/hash
input/output schema
job definition summary
validation status
source_yaml optional
```

Definition readでWorkflow Runを作成しない。

## 6. Workflow Run operations

### `wf_start`

request:

```text
workflow_ref
inputs
priority optional
source_identity optional
request_id optional
```

response:

```text
workflow_run_id
status
run_attempt
created_at
```

validation failureはRun rowを作らない。

### `wf_run_list`

filter:

```text
workflow_id
status
conclusion
created_from/to
parent_workflow_run_id optional
limit/cursor
```

### `wf_run_info`

request:

```text
workflow_run_id
include_jobs=false
include_attempts=false
include_steps=false
include_artifacts=false
include_events_summary=false
```

`include_attempts/steps/artifacts`がtrueなら必要な上位情報をAdapter/Serviceが自動包含できる。

Log本文は含めない。log availability/size/latest timeはsummary可能。

## 7. Workflow Run control

```text
wf_pause
wf_resume
wf_cancel
wf_retry
wf_priority_update
```

すべてrequest_id optional。

### repeat semantics

- pause on paused -> no-op success
- resume on non-paused active Run -> no-op success
- cancel on already cancelled Run -> no-op success
- pause/resume on completed -> conflict
- cancel on completed success/failure -> conflict（既にcancelledならno-op）
- retryは`10`のfailed Job指定条件のみ

### Child Run direct control

`parent_workflow_run_id != null`のRunへのpublic pause/resume/cancel/retry/priority updateは:

```text
child_run_direct_control_forbidden
```

Readは許可。Parent propagationのinternal operationは別。

### `wf_retry`

request:

```text
workflow_run_id
job_run_id
request_id optional
```

Job-only。Input変更fieldは存在しない。completed failed Runは`10`に従いreopen。

### `wf_priority_update`

request:

```text
workflow_run_id
priority integer
request_id optional
```

実行中Jobはpreemptしない。

## 8. External Task operations

### `wf_task_info`

read-only。task_idまたはcurrent Job/Attemptから取得。

### `wf_task_claim`

request:

```text
claimed_by
workflow_run_id optional
job_key optional
request_id optional
```

response:

```text
task_id/lease_id/lease_expires_at
workflow_run_id/job_run_id/attempt_id
input
metadata
```

### `wf_task_submit`

request:

```text
task_id
lease_id
result
artifacts optional
claim_next=false
request_id optional
```

`artifacts`は保存済み実体へのmetadata reference。Result/Artifact/Attempt/Job/Event/idempotencyをatomicに確定。`claim_next`だけはsubmit commit後の別claim transaction。

## 9. Human Review discovery/control

### `wf_review_list`

filter:

```text
workflow_run_id optional
status=pending|completed|cancelled optional
limit/cursor
```

### `wf_review_info`

request `review_id`。Review Input、Artifact refs、status/outcome/comment/actor summaryを返せる。

### `wf_review_submit`

```text
review_id
outcome approve|reject
comment optional
request_id optional
```

first terminal submit wins。

## 10. Artifact / Log / Runner read

### `wf_artifact_info`

artifact_idまたはworkflow/job/name filter。Metadata/historyのみ。Core標準download無し。

### `wf_log_read`

```text
attempt_id
offset optional
limit optional
tail_lines optional
```

返却:

```text
content
next_offset
truncated
size_bytes
updated_at
```

### `wf_runner_info`

Pool/Runner status, instance ID, heartbeat, current Job, restart count/suppression。

Public force-kill無し。

## 11. MCP public names

Core logical nameの先頭へ`<system_namespace>_`を付ける。

例:

```text
novel_wf_definition_list
novel_wf_definition_info
novel_wf_start
novel_wf_run_list
novel_wf_run_info
fx_wf_task_claim
```

MVP tool set:

```text
<ns>_wf_definition_list
<ns>_wf_definition_info
<ns>_wf_start
<ns>_wf_run_list
<ns>_wf_run_info
<ns>_wf_pause
<ns>_wf_resume
<ns>_wf_cancel
<ns>_wf_retry
<ns>_wf_priority_update
<ns>_wf_task_info
<ns>_wf_task_claim
<ns>_wf_task_submit
<ns>_wf_review_list
<ns>_wf_review_info
<ns>_wf_review_submit
<ns>_wf_artifact_info
<ns>_wf_log_read
<ns>_wf_runner_info
```

`system_namespace`は空禁止。推奨syntax `^[a-z][a-z0-9_]*$`。親システムは不要toolを登録しない選択ができる。

## 12. MCP granularity

万能tool 1個へ集約しない。Read/Run control/External/Human/Logを分離し、Agentが意図しないstate changeを起こしにくいschemaにする。

## 13. Pagination

list/historyはcursor pagination。外部contract:

```text
items
next_cursor nullable
```

Cursorはopaque。Default limit 50、max 200をMVP既定としSystemで下げてもよい。

## 14. Error model

```text
code
message
retryable
field/path optional
details optional
```

共通code:

```text
not_found
validation_failed
conflict
unauthorized
forbidden
invalid_state
idempotency_conflict
lease_conflict
lease_expired
runner_unavailable
child_run_direct_control_forbidden
internal_error
```

MCPはstructured tool error、WebはAdapterがHTTPへmapping。

## 15. Idempotency order

state-changing request:

1. request schema
2. parent-derived ActorContext
3. Authorization
4. canonical request hash作成（Secret value無し）
5. idempotency record lookup
6. same key/hash completedならstored response replay
7. different hashならconflict
8. domain validation
9. transactionでside effect + Event + idempotency result
10. response

Concurrent same keyはDB uniqueness/transactionによりsingle side effect。

Default TTL 24hは`08`。

## 16. Python API

同じtyped request/response modelを使う。

```python
run = services.workflow_runs.start(
    workflow_ref="analysis",
    inputs={"document_id": 1},
    actor=actor,
)
```

Python APIだからAuthorizationを迂回することはない。

## 17. Web Adapter

画面詳細は別途。HTTP AdapterはService modelの薄いmapping。

Exact URL/path/versioningはWeb/API設計で決められるが、**Service operation名と意味を変えない**。

## 18. Internal Runner Service

Runner Processが使う:

```text
runner_register/heartbeat/claim/attempt reflection
```

はpublic MCP/Web operationではない。Runtime/Runner identity fencingを使う内部API。Public Actor Authorizationとは別のruntime trust boundaryだが、DB state transitionは同じRepository規則を使う。

## 19. Observability

state change Eventに:

```text
actor/source/request_id
```

を保存可能。Request body全体やSecretを無条件保存しない。

## 20. 受入条件

1. Definition list/infoとRun list/infoの区別
2.全read/write Authorization
3. MCP namespace/tool set
4. wf_run_info include hierarchy
5. Log separate read
6. pause/resume/cancel repeat semantics
7. Manual Retry reopen
8. priority update
9. Child public control拒否/read許可
10. task submit Artifact + claim_next
11. Review list/info/submit
12. idempotency side effect同transaction/concurrent same key
13. cursor default/max
14. parent tool非公開選択
15. Python/MCP/Web contract equivalence
