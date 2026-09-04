# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v0.8
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `07`, `08`, `09`, `10`, `12`

## 1. 基本原則

1. MCP/HTTP/Python APIは同じService layer。
2. AdapterはDB/Storeを直接更新しない。
3. Public read/writeはAuthorizationProviderを通す。
4. State-changing operationはoptional `request_id`。
5. MCP public tool名は親namespace必須。
6. Large Log/Output本文をRun infoへ埋め込まない。
7. Workflow DefinitionとWorkflow RunをAPI名で分離。
8. Service request/response modelを本書で固定し、Adapter独自fieldをCore意味へ持ち込まない。
9. Job状態を任意に書き換えるgeneric mutation APIを提供しない。

## 2. 共通model規則

全request/responseはPydantic typed model + JSON-compatible serialization。

- IDはCore生成opaque string。
- `ActorContext/AccessScope` はAdapter/parentからServiceへ別argumentで注入し、public request body/tool schemaへcaller自由入力として露出しない。
- `request_id` はPython/MCP requestではoptional string、HTTPでは`Idempotency-Key` headerから注入。
- Unknown request fieldはreject。
- TimestampはRFC3339 UTC string。

### 2.1 Pagination

List request共通:

```text
limit: integer 1..200, default 50
cursor: opaque string optional
```

List response:

```json
{
  "items": [...],
  "next_cursor": null
}
```

Orderingはoperationごとに固定し、cursorにはordering key + ID tie-breakをopaque encodingする。

### 2.2 Mutation summary

Run state change共通response:

```text
workflow_run_id
status
conclusion nullable
updated_at
```

## 3. Service構成

```text
WorkflowDefinitionService
WorkflowRunService
JobService
OutputService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
RunnerService
```

## 4. Workflow Definition operations

### `wf_definition_list`

Request=`limit/cursor`。

Order: canonical `workflow_id` ASC。

Item:

```text
workflow_id
name
version
description nullable
definition_hash
source_kind
source_display nullable
```

### `wf_definition_info`

Request:

```text
workflow_ref: non-empty string
include_source_yaml: boolean default false
include_jobs: boolean default true
```

Response:

```text
workflow_id
name
version
description nullable
definition_hash
source_kind/source_display
inputs_schema
outputs_definition
jobs nullable
source_yaml nullable
```

HTTPはreferenceをqueryへ渡す。Path segmentへ埋め込まない。

## 5. Workflow Run operations

### 5.1 `wf_start`

Request:

```text
workflow_ref: non-empty string
inputs: object default {}
priority: signed64 integer optional
source_identity: non-empty string optional
request_id optional
```

Response:

```text
workflow_run_id
workflow_id
status
conclusion nullable
wait_reason nullable
created_at
```

Concurrency `on-limit=queue`でもRunを作りstatus/wait_reasonを返す。`reject`はerrorでRun ID無し。

### 5.2 `wf_run_list`

Request:

```text
workflow_id optional
status optional
conclusion optional
created_from optional
created_to optional
limit/cursor
```

Order: `created_at DESC, workflow_run_id ASC`。

Item:

```text
workflow_run_id
workflow_id
workflow_version
status/conclusion
priority
run_attempt
wait_reason
created_at/started_at/completed_at
```

### 5.3 `wf_run_info`

Request:

```text
workflow_run_id
include_jobs=false
include_attempts=false
include_steps=false
include_artifacts=false
include_events_summary=false
include_output_metadata=true
```

`include_attempts=true` は `include_jobs`を暗黙true、`include_steps=true`はjobs+attemptsを暗黙true。

Response base:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
pause_requested/cancel_requested
source_identity nullable
created_at/started_at/completed_at
output_metadata nullable
jobs nullable
events_summary nullable
```

Output/Log本文無し。

### 5.4 `wf_pause`

Request=`workflow_run_id + request_id?`。

Allowed=non-terminal root Run。Already pausedはsame state idempotent success。Child direct control reject。

### 5.5 `wf_resume`

Allowed=paused root Run。Already non-paused non-terminalへの新requestは`invalid_state`。同idempotency key replayは元resultを返す。

### 5.6 `wf_cancel`

Request:

```text
workflow_run_id
reason optional string
request_id optional
```

Non-terminal root Run。Already cancelled terminalでsame request_idならreplay。別requestは`invalid_state`。

### 5.7 `wf_priority_update`

Request:

```text
workflow_run_id
priority: signed64 integer
request_id optional
```

Root non-terminal Runのみ。Preempt無し。Response=Mutation summary + priority。

### 5.8 `wf_retry`

Request:

```text
workflow_run_id
job_run_id
request_id optional
```

Eligibility=`10`。Input override fieldをschemaに持たない。

Response:

```text
workflow_run_id
job_run_id
run_attempt
job_status
retry_not_before nullable
updated_at
```

Manual Retry request時点ではnew Attemptを作らないためattempt_id無し。

## 6. Output operations

### `wf_output_info`

Request exactly one:

```text
workflow_run_id xor job_run_id xor attempt_id
```

Response:

```text
source_type=workflow_run|job_run|attempt
source_id
available
storage_kind nullable
size_bytes nullable
digest nullable
```

### `wf_output_read`

同じselector + optional `select: JMESPath string`。

Response:

```text
source_type
source_id
selected: boolean
value: any JSON-compatible value
```

Output unavailable=`not_found`。Payload corruption=storage error。MCP上限超過は`response_too_large`、silent truncate無し。

## 7. External Task operations

### 7.1 `wf_task_info`

Request=`task_id`。

Response:

```text
task_id/workflow_run_id/job_run_id/attempt_id
status
input
current_lease nullable
claim_history_summary
output_metadata nullable
failure nullable
created_at/completed_at
```

### 7.2 `wf_task_claim`

Request:

```text
task_id optional
workflow_run_id optional
job_template_key optional
request_id optional
```

`task_id`指定時は他filter禁止。未指定時は通常priority orderingでcandidateを選ぶ。

Claimant identityはActorContext/client principalからCoreが作り、自由な`claimed_by` field無し。

候補無し:

```json
{"task": null}
```

Success task:

```text
task_id
lease_id
lease_expires_at
workflow_run_id
job_run_id
attempt_id
job_key
input
```

### 7.3 `wf_task_submit`

Request:

```text
task_id
lease_id
result: any JSON-compatible value
artifacts: array default []
claim_next: boolean default false
request_id optional
```

Artifact item:

```text
name
uri
media_type optional
size_bytes optional
digest optional
metadata optional object
```

Response:

```text
submitted=true
workflow_run_id/job_run_id/attempt_id
job_status/job_conclusion
failure nullable
next_task nullable
```

Validation/domain failureをterminal failureとして受理した場合も`submitted=true`。Lease conflict/stale/cancelはoperation error。

`claim_next=true` のnext claim失敗/候補無しはsubmit本体をrollbackせず`next_task=null`。

Idempotency replayでは初回submit responseをそのまま再生するため、初回に返した`next_task`も同じresponseとして返す。Replay時に別Taskを追加claimしない。

### 7.4 MVPに存在しないLease操作

以下のService operationはMVPに**存在しない**。

```text
wf_task_heartbeat
wf_task_lease_renew
wf_task_lease_extend
wf_task_lease_transfer
```

対応するMCP tool/HTTP endpointも作らない。Lease lifetime変更はDefinition/System設定の次Attempt/Taskへ適用する。

## 8. Human Review operations

### `wf_review_list`

Request:

```text
workflow_run_id optional
status optional=pending|completed|cancelled
limit/cursor
```

Order=`created_at ASC, review_id ASC`。

### `wf_review_info`

Request=`review_id`。Review/input/outcome/comment/actor summary/timestampsを返す。

### `wf_review_submit`

Request:

```text
review_id
outcome: approve|reject
comment optional string
request_id optional
```

Response:

```text
review_id
workflow_run_id/job_run_id/attempt_id
status=completed
outcome
job_status/job_conclusion
completed_at
```

Completed/cancelled Reviewへの別outcome再submitは`invalid_state`。Same idempotency keyだけreplay可能。

## 9. Artifact / Log / Runner operations

### `wf_artifact_info`

Request=`artifact_id`。Response=canonical public ArtifactRef + producer IDs + created/deleted metadata。Managed store_key/path無し。

Cross-run readもAuthorization対象。

### `wf_log_read`

Request:

```text
attempt_id
offset_bytes optional >=0
limit_bytes optional 1..1048576 default 65536
tail_lines optional 1..10000
```

`tail_lines`と`offset_bytes`同時禁止。

### `wf_runner_info`

Request=`pool optional`。Response=`pools[{name,configured_count,runners[]}]`。

## 10. 禁止するgeneric Job mutation

MVP Public Serviceに以下を作らない。

```text
wf_job_mark_success
wf_job_override_conclusion
wf_job_skip
wf_job_force_complete
wf_review_rewrite
```

したがってMCP/HTTPにも対応tool/route無し。

- failed Jobをsuccessにしたい -> 原因修正後`wf_retry`
- skip/許容failure -> Workflow Definitionの`if`/`continue-on-error`等で事前定義
- Human Review -> pending状態へのapprove/rejectのみ

Core内部Recoveryもterminal success/failureを「管理者操作だから」という理由で上書きしない。

## 11. MCP namespace / canonical tools

`system_namespace` non-empty、推奨 `^[a-z][a-z0-9_]*$`。同一MCP server tool collisionをregistration時reject。

Canonical public tools:

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
<ns>_wf_output_info
<ns>_wf_output_read
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

Actor/AccessScopeはpublic tool inputへ露出しない。

## 12. HTTP Adapter v1

Standard prefix=`/api/jobrunner/v1`。

```text
GET  /workflow-definitions
GET  /workflow-definitions/info?workflow_ref=<encoded>
POST /workflow-runs
GET  /workflow-runs
GET  /workflow-runs/{workflow_run_id}
POST /workflow-runs/{workflow_run_id}/pause
POST /workflow-runs/{workflow_run_id}/resume
POST /workflow-runs/{workflow_run_id}/cancel
PATCH /workflow-runs/{workflow_run_id}
POST /workflow-runs/{workflow_run_id}/jobs/{job_run_id}/retry
GET  /workflow-runs/{workflow_run_id}/output-info
GET  /workflow-runs/{workflow_run_id}/output
GET  /jobs/{job_run_id}/output-info
GET  /jobs/{job_run_id}/output
GET  /attempts/{attempt_id}/output-info
GET  /attempts/{attempt_id}/output
GET  /external-tasks/{task_id}
POST /external-tasks/claim
POST /external-tasks/{task_id}/submit
GET  /reviews
GET  /reviews/{review_id}
POST /reviews/{review_id}/submit
GET  /artifacts/{artifact_id}
GET  /attempts/{attempt_id}/log
GET  /runners
```

Opaque generated IDのみpath parameter。Workflow referenceはquery/body。

Path IDをService fieldへinjectし、bodyに同IDを重複要求しない。GET queryは対応read modelへmapping。

State-changing HTTPはoptional `Idempotency-Key` header -> Service request_id。Bodyにrequest_id無し。

Lease renew/heartbeat、Job skip/override用routeは存在しない。

## 13. HTTP status mapping

```text
200  read / successful non-create mutation
201  new top-level Workflow Run creation
400  request/definition/input/expression/domain contract validation
401  unauthenticated
403  forbidden
404  not_found policy
409  state/idempotency/lease ownership conflict
413  explicit Adapter request/response size policy
500  internal_error
```

422はMVP不使用。

Idempotency replayは最初の成功status+bodyをそのまま再生。Persistence `adapter_meta_json`にoriginal statusを保持可能。

## 14. Error model

```text
code
message
retryable
field/path optional
details optional
```

代表:

```text
not_found
validation_failed
conflict
unauthorized
forbidden
invalid_state
retry_input_unavailable
action_version_mismatch
validator_version_mismatch
idempotency_conflict
lease_conflict
lease_expired
runner_unavailable
payload_missing
payload_digest_mismatch
artifact_access_forbidden
artifact_data_unavailable
response_too_large
child_run_direct_control_forbidden
internal_error
```

## 15. Idempotency

対象:

```text
wf_start/wf_pause/wf_resume/wf_cancel/wf_retry/wf_priority_update
wf_task_claim/wf_task_submit/wf_review_submit
```

Identity=`scope + operation + request_id`。

Scope=namespace+resource+AccessScope+Actor/client principal。

TTL内same hash replay/different hash conflict。TTL後expired row replace可。

Persisted Service result + Adapter original HTTP status metadata。Replayで副作用を再実行しない。

## 16. Validation order

1. request schema
2. Actor/AccessScope
3. Authorization
4. idempotency lookup/reservation
5. domain validation
6. side effect + Event + result transaction
7. response

Secret値をrequest hash/Eventへ保存しない。

## 17. Authorization / pagination

全public operation authorize。Listはfiltered scope適用。

Cursor opaque。`limit`共通1..200 default50。

## 18. Python API

Canonical Service request/response modelを直接使う。Adapter独自意味変更禁止。

## 19. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payload無条件保存無し。

## 20. 受入条件

1. Service request unknown field reject
2. Actor/AccessScope caller injection禁止
3. pagination exact model/order
4. Definition/Run operation separation
5. start/list/info response shapes
6. state mutation request/response
7. retry no input override/no attempt id response
8. output xor selector
9. task specific/discovery claim + no-candidate null
10. task submit idempotency replay does not claim new next task
11. no lease heartbeat/renew/extend/transfer operation/tool/route
12. review submit completed rewrite reject
13. no manual Job skip/success override generic mutation
14. artifact access errors/authorization
15. log read parameter conflicts/ranges
16. namespaced MCP collision
17. HTTP path/body ID mapping
18. slash-containing workflow_ref query
19. Idempotency-Key/original status replay
20. no 422
21. all read/write authorization
