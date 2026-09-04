# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `07`, `08`, `09`, `10`, `12`

## 1. 基本原則

1. MCP/HTTP/Pythonは同じService layer。
2. AdapterはDB/Store直接更新禁止。
3. Public read/writeはAuthorizationProvider必須。
4. State-changing operationはoptional `request_id`。
5. MCP toolは親namespace必須。
6. Large Log/Output本文をRun infoへ埋め込まない。
7. Workflow DefinitionとWorkflow RunをAPI名で分離。
8. Canonical request/response modelを本書で固定。
9. Generic Job state override API無し。

## 2. 共通model

Pydantic typed model + JSON-compatible serialization。

- Core ID=opaque string
- ActorContext/AccessScopeはAdapter/parentから別argument注入。Caller自由入力にしない
- Python/MCP `request_id` optional
- HTTPは`Idempotency-Key` header -> request_id
- Unknown request field reject
- Timestamp=RFC3339 UTC

### Pagination

```text
limit integer 1..200 default50
cursor opaque optional
```

Response:

```json
{"items":[],"next_cursor":null}
```

## 3. Service components

```text
WorkflowDefinitionService
WorkflowRunService
OutputService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
EventService
RunnerService
```

## 4. Workflow Definition

### `wf_definition_list`

Request=`limit/cursor`。Order=`workflow_id ASC`。

Item:

```text
workflow_id/name/version/description
definition_hash/source_kind/source_display
```

### `wf_definition_info`

Request:

```text
workflow_ref non-empty
include_source_yaml=false
include_jobs=true
```

Response:

```text
workflow_id/name/version/description
definition_hash
source_kind/source_display
inputs_schema
outputs_definition
jobs nullable
source_yaml nullable
```

HTTPはworkflow_refをqueryへ。

## 5. Workflow Run

### `wf_start`

Request:

```text
workflow_ref non-empty
inputs object default {}
priority signed64 optional
source_identity non-empty optional
request_id optional
```

Response:

```text
workflow_run_id
workflow_id
status/conclusion
wait_reason
created_at
```

Run start時`01/08` effective settings/Retention/Registry version snapshotを固定。

Concurrency queueでもRun作成。RejectはRun ID無しerror。

### `wf_run_list`

Filters:

```text
workflow_id/status/conclusion/created_from/created_to optional
limit/cursor
```

Order=`created_at DESC,workflow_run_id ASC`。

Item:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
progress nullable
created_at/started_at/completed_at
```

### `wf_run_info`

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

Attempts implies jobs、steps implies jobs+attempts。

Base Response:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
pause_requested/cancel_requested
source_identity nullable
progress nullable
created_at/started_at/completed_at
output_metadata nullable
jobs nullable
events_summary nullable
```

Job itemは少なくとも:

```text
job_run_id/job_key/job_template_key
executor/status/conclusion
priority/progress
current_attempt_id
failure nullable
```

Output/Log本文無し。

### `wf_pause`

Non-terminal root Run。Already paused=same-state success。

### `wf_resume`

Paused root Run。New requestでnon-pausedならinvalid_state。Replayはstored result。

### `wf_cancel`

Request=`workflow_run_id,reason?,request_id?`。Non-terminal root only。

### `wf_priority_update`

Request=`workflow_run_id,priority,request_id?`。Root non-terminal。Preempt無し。

### `wf_retry`

Request:

```text
workflow_run_id
job_run_id
request_id optional
```

Input override field無し。

Response:

```text
workflow_run_id/job_run_id
run_attempt
job_status
retry_not_before nullable
updated_at
```

Request時点ではnew Attempt無し。

## 6. Output

### `wf_output_info`

Exactly one selector:

```text
workflow_run_id xor job_run_id xor attempt_id
```

Response=`source_type/source_id/available/storage_kind/size_bytes/digest`。

### `wf_output_read`

Same selector + optional JMESPath `select`。

Response:

```text
source_type/source_id
selected boolean
value any JSON
```

MCP response上限超過=silent truncateせず`response_too_large`。

## 7. External Task

### `wf_task_info`

Request=`task_id`。

Response:

```text
task_id/workflow_run_id/job_run_id/attempt_id
status/input
current_lease nullable
claim_history_summary
output_metadata/failure nullable
created_at/completed_at
```

### `wf_task_claim`

Request:

```text
task_id optional
workflow_run_id optional
job_template_key optional
request_id optional
```

Task ID指定時他filter禁止。No candidate=`{"task":null}`。

Task object:

```text
task_id/lease_id/lease_expires_at
workflow_run_id/job_run_id/attempt_id/job_key
input
```

ClaimantはActor/client principalからCore生成。

### `wf_task_submit`

Request:

```text
task_id
lease_id
result any JSON
artifacts array default[]
claim_next boolean defaultfalse
request_id optional
```

Artifact item=`name,uri,media_type?,size_bytes?,digest?,metadata?`。

Response:

```text
submitted=true
workflow_run_id/job_run_id/attempt_id
job_status/job_conclusion
failure nullable
next_task nullable
```

Validation failureをterminal failureとして受理した場合もsubmitted=true。

`claim_next=true` は`07/08`どおり**submit terminal state + optional next Lease + full response/idempotency resultを同一DB transaction**で確定する。No candidateはnext_task=null。Replay時new Taskをclaimしない。

### Lease非対応API

無し:

```text
wf_task_heartbeat
wf_task_lease_renew
wf_task_lease_extend
wf_task_lease_transfer
```

## 8. Human Review

### `wf_review_list`

Filters=`workflow_run_id?,status?,limit,cursor`。Order=`created_at ASC,review_id ASC`。

### `wf_review_info`

Request=`review_id`。Input/outcome/comment/actor/timestampsを返す。

### `wf_review_submit`

Request=`review_id,outcome approve|reject,comment?,request_id?`。

Completed/cancelled Review rewrite不可。Same idempotency key replayのみ元response。

## 9. Artifact / Log / Event / Runner

### `wf_artifact_info`

Request=`artifact_id`。Response=canonical public ArtifactRef + producer/deletion metadata。Store path無し。Cross-run read Authorization必須。

### `wf_log_read`

Request:

```text
attempt_id
offset_bytes>=0 optional
limit_bytes 1..1048576 default65536
tail_lines 1..10000 optional
```

Tailとoffset同時禁止。

Response:

```text
content
next_offset_bytes nullable
truncated
size_bytes
updated_at nullable
```

Deleted log=`log_data_unavailable`。External path指定不可。

### `wf_event_list`

Request:

```text
workflow_run_id optional
job_run_id optional
attempt_id optional
types array<non-empty string> optional
created_from/created_to optional
include_payload boolean default true
limit/cursor
```

No owner filterの場合もAuthorizationProviderによるfiltered scopeを必須とする。

Order=`created_at DESC,event_id DESC`。

Item:

```text
event_id/type/version
workflow_run_id/job_run_id/attempt_id nullable
runner_id nullable
actor_type/actor_id/source/request_id nullable
payload nullable
created_at
```

System retention audit Eventも権限policyに従い取得可能。

### `wf_runner_info`

Request=`pool?`。Response=`pools[{name,configured_count,runners[]}]`。

## 10. 禁止generic mutation

無し:

```text
wf_job_mark_success
wf_job_override_conclusion
wf_job_skip
wf_job_force_complete
wf_review_rewrite
```

Failed Jobは原因修正後Retry。Skip/許容failureはDefinitionで事前定義。

## 11. MCP namespace

`system_namespace` non-empty、推奨`^[a-z][a-z0-9_]*$`。Tool collision registration時reject。

Canonical tools:

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
<ns>_wf_event_list
<ns>_wf_runner_info
```

Actor/AccessScopeはtool inputへ露出しない。

## 12. HTTP Adapter v1

Prefix=`/api/jobrunner/v1`。

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
GET  /events
GET  /runners
```

Opaque Core IDsのみpath parameter。Workflow ref=query/body。

Path IDはAdapterがService fieldへinject。Body重複要求無し。

State-changing HTTP optional `Idempotency-Key` header。Body request_id無し。

Lease renew/heartbeat/Job skip/override route無し。

## 13. HTTP status

```text
200 read / successful non-create mutation
201 top-level Workflow Run creation
400 request/definition/input/expression/domain validation
401 unauthenticated
403 forbidden
404 not_found
409 state/idempotency/lease conflict
413 explicit Adapter size policy
500 internal_error
```

422無し。Idempotency replayはoriginal successful status/body。

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
unauthorized/forbidden
invalid_state
retry_input_unavailable
action_version_mismatch
validator_version_mismatch
idempotency_conflict
lease_conflict/lease_expired
runner_unavailable
payload_missing/payload_digest_mismatch
artifact_access_forbidden/artifact_data_unavailable
log_data_unavailable
secret_binding_invalid
successful_job_result_not_reusable
dynamic_expansion_not_reusable
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

`request_hash`=`08` canonical Service request excluding request_id/transport fields。

Flow:

1. request schema
2. Actor/AccessScope
3. Authorization
4. optional fast idempotency lookup
5. domain/expensive validation + filesystem temp prepare
6. `BEGIN IMMEDIATE`
7. idempotency key/hash **再確認**
8. current domain state再確認
9. side effect + full Service result + completed idempotency row commit
10. Adapter response

DB `reserved` rowは作らない。Concurrent same-keyはSQLite write serialization + commit-time recheckで片方だけside effect commit。

TTL内same hash=replay、different hash=conflict。Expired row replace可。

`wf_task_submit(claim_next=true)`のnext Leaseも同transaction/idempotency result内。

## 16. Authorization / pagination

全public operation authorize。List/Eventはfiltered scope。Cursor opaque。Limit1..200 default50。

## 17. Python API

Canonical Service request/response modelを直接使用。Adapter独自意味変更禁止。

## 18. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 19. 受入条件

1. Definition/Run separation
2. request unknown field/Actor injection reject
3. pagination/order
4. Run info progress + nested include semantics
5. retry no Input override/no Attempt creation
6. Output selector/read
7. task claim no-candidate null
8. submit+claim_next same transaction/replay
9. no lease renew APIs
10. Human rewrite/manual Job mutation無し
11. Artifact/Log errors
12. `wf_event_list` filters/order/Authorization
13. namespaced MCP collision
14. HTTP exact routes/Idempotency-Key/status/no422
15. idempotency canonical hash/no-reserved/commit recheck
16. all read/write Authorization
