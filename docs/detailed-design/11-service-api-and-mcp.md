# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v0.7
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `07`, `08`, `10`, `12`

## 1. 基本原則

1. MCP/HTTP/Python APIは同じService layer。
2. AdapterはDB/Storeを直接更新しない。
3. Public read/writeはAuthorizationProviderを通す。
4. State-changing operationはoptional `request_id`。
5. MCP public tool名は親namespace必須。
6. Large Log/Output本文をRun infoへ埋め込まない。
7. Workflow DefinitionとWorkflow RunをAPI名で分離。
8. Service request/response modelを本書で固定し、Adapter独自fieldをCore意味へ持ち込まない。

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

Request:

```text
limit/cursor
```

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

`include_attempts=true` は `include_jobs`を暗黙true、`include_steps=true`はjobs+attemptsを暗黙trueにする。

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

Request: `workflow_run_id + request_id?`。

Allowed: non-terminal root Run。Already pausedはsame state idempotent success。Child direct control reject。

Response: Mutation summary。

### 5.5 `wf_resume`

Allowed: paused root Run。Already non-paused non-terminalへの再送は、同idempotency key replay以外 `invalid_state`。Response Mutation summary。

### 5.6 `wf_cancel`

Request:

```text
workflow_run_id
reason optional string
request_id optional
```

Non-terminal root Run。Already cancelled terminalでsame request_idならreplay。異なるnew cancel requestは`invalid_state`。

Response Mutation summary。

### 5.7 `wf_priority_update`

Request:

```text
workflow_run_id
priority: signed64 integer
request_id optional
```

Root non-terminal Runのみ。Preempt無し。Response Mutation summary + `priority`。

### 5.8 `wf_retry`

Request:

```text
workflow_run_id
job_run_id
request_id optional
```

Eligibilityは`10`。Input override field自体をschemaに持たない。

Response:

```text
workflow_run_id
job_run_id
run_attempt
job_status
retry_not_before nullable
updated_at
```

Manual Retry request時点ではnew Attemptを作らないため`attempt_id`を返さない。

## 6. Output operations

### `wf_output_info`

Request exactly one source selector:

```text
workflow_run_id xor job_run_id xor attempt_id
```

Response:

```text
source_type = workflow_run|job_run|attempt
source_id
available
storage_kind nullable
size_bytes nullable
digest nullable
```

### `wf_output_read`

同じsource selector +:

```text
select: JMESPath string optional
```

Response:

```text
source_type
source_id
selected: boolean
value: any JSON-compatible value
```

Output unavailableは`not_found`。Payload corruptionはstorage error。

MCP response上限超過時silent truncateせず`response_too_large`。

## 7. External Task operations

### 7.1 `wf_task_info`

Request: `task_id`。

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
task_id optional                 # specific available task
workflow_run_id optional         # discovery filter
job_template_key optional        # discovery filter
request_id optional
```

`task_id`指定時は他discovery filter禁止。未指定時はfilterで次candidateを選ぶ。

Claimant identityはActorContext/client principalからCoreが作り、request bodyの自由な`claimed_by`は受けない。

候補無しはerrorにせず:

```json
{"task": null}
```

Success:

```json
{
  "task": {
    "task_id": "...",
    "lease_id": "...",
    "lease_expires_at": "...",
    "workflow_run_id": "...",
    "job_run_id": "...",
    "attempt_id": "...",
    "job_key": "...",
    "input": {}
  }
}
```

Inputは任意JSON-compatible object（Job Input contract上object）。Secret無し。

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
next_task nullable       # same shape as claim task object
```

Validation/domain failureがsubmitをterminal failureとして受理した場合も`submitted=true`でfailureを返す。Lease conflict/stale/cancelはoperation errorで`submitted=false` responseを作らない。

`claim_next=true` のnext claim失敗/候補無しはsubmit本体をrollbackせず`next_task=null`。

## 8. Human Review operations

### `wf_review_list`

Request:

```text
workflow_run_id optional
status optional = pending|completed|cancelled
limit/cursor
```

Order: `created_at ASC, review_id ASC`。

Item:

```text
review_id/workflow_run_id/job_run_id/attempt_id
status/outcome nullable
created_at/completed_at
```

### `wf_review_info`

Request: `review_id`。

Response:

```text
review_id/workflow_run_id/job_run_id/attempt_id
status/outcome/comment
input
actor_summary nullable
created_at/completed_at
```

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

## 9. Artifact / Log / Runner operations

### `wf_artifact_info`

Request: `artifact_id`。

Response=public ArtifactRef + producer IDs + created/deleted metadata。Managed `store_key`/filesystem pathは返さない。

### `wf_log_read`

Request:

```text
attempt_id
offset_bytes optional >=0
limit_bytes optional 1..1048576 default 65536
tail_lines optional 1..10000
```

`tail_lines` と `offset_bytes` は同時指定禁止。Response:

```text
content string
next_offset_bytes nullable
truncated boolean
size_bytes
updated_at nullable
```

### `wf_runner_info`

Request:

```text
pool optional non-empty string
```

Response:

```text
pools: [{name, configured_count, runners:[...]}]
```

Runner itemはrunner/instance/runtime ID, status, heartbeat, current job/attempt, restart/suppression summary。

## 10. MCP namespace / canonical tools

`system_namespace` non-empty、推奨:

```text
^[a-z][a-z0-9_]*$
```

同一MCP server tool collisionをregistration時reject。

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

MCP schemaは上記Service requestから`request_id`を含める以外、Actor/AccessScopeを除いた同じfield意味を使う。

## 11. HTTP Adapter v1

Standard prefix `/api/jobrunner/v1`。

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

上記はprefix付きで解釈。Opaque generated IDのみpath parameter。Workflow referenceはquery/body。

HTTP Adapterはpathから取得したIDをService request fieldへinjectし、bodyに同じID fieldを重複要求しない。

GET queryは対応Service read request fieldsへmapping。

State-changing HTTPはoptional `Idempotency-Key` header -> Service request_id。Bodyにrequest_id無し。

## 12. HTTP status mapping

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

### Idempotency replay

最初の成功status+bodyをそのまま再生。

- wf_start初回201 -> replay201
- pause初回200 -> replay200

Persistence `adapter_meta_json`にoriginal HTTP statusを保持可能。

MCP`response_too_large`はtool error。HTTPが独自response上限を設定して拒否するなら413。

## 13. Error model

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
response_too_large
child_run_direct_control_forbidden
internal_error
```

## 14. Idempotency

対象:

```text
wf_start/wf_pause/wf_resume/wf_cancel/wf_retry/wf_priority_update
wf_task_claim/wf_task_submit/wf_review_submit
```

Identity=`scope + operation + request_id`。

Scope=namespace+resource+AccessScope+Actor/client principal。

TTL内same hash replay/different hash conflict。TTL後expired row replace可。

Persisted Service result + Adapter original HTTP status metadata。MCP/Pythonは同じService resultを各形式へ再構成。

## 15. Validation order

1. request schema
2. Actor/AccessScope
3. Authorization
4. idempotency lookup/reservation
5. domain validation
6. side effect + Event + result transaction
7. response

Secret値をrequest hash/Eventへ保存しない。

## 16. Authorization / pagination

全public operation authorize。Listはfiltered scope適用。

Cursor opaque。`limit`共通1..200 default50。

## 17. Python API

Canonical Service request/response modelを直接使う。Adapter独自意味変更禁止。

## 18. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payload無条件保存無し。

## 19. 受入条件

1. Service request unknown field reject
2. Actor/AccessScope caller injection禁止
3. pagination exact model/order
4. Definition/Run operation separation
5. start/list/info response shapes
6. state mutation request/response
7. retry no input override/no attempt id response
8. output xor selector
9. task specific/discovery claim + no-candidate null
10. task submit terminal validation failure semantics
11. review request/response
12. log read parameter conflicts/ranges
13. namespaced MCP collision
14. HTTP path/body ID mapping
15. slash-containing workflow_ref query
16. Idempotency-Key/original status replay
17. no 422
18. all read/write authorization
