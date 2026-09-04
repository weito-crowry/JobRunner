# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v1.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `07`, `08`, `09`, `10`, `12`

## 1. 基本原則

1. MCP/HTTP/Pythonは同じService layer。
2. AdapterはDB/Store直接更新禁止。
3. Public read/writeはAuthorizationProvider必須。
4. State-changing operationはoptional `request_id`。
5. MCP toolは親namespace必須。
6. Large Input/Output/Log本文をRun infoへ埋め込まない。
7. Workflow DefinitionとWorkflow RunをAPI名で分離。
8. Canonical request/response modelを本書で固定。
9. Generic Job state override API無し。

## 2. 共通model

Pydantic typed model + JSON-compatible serialization。

- Core ID=opaque string
- ActorContext/AccessScopeはAdapter/parentから別argument注入
- Python/MCP `request_id` optional
- HTTPは`Idempotency-Key` header -> request_id
- Unknown request field reject
- Timestamp=`08` canonical UTC fixed form `YYYY-MM-DDTHH:MM:SS.ffffffZ`

Timestamp input filterも同形式へparse/normalizeしてからServiceへ渡す。Naive/local offset文字列をcanonical responseへそのままechoしない。

Pagination:

```text
limit integer 1..200 default50
cursor opaque optional
```

Response=`{"items":[],"next_cursor":null}`。

## 3. Service components

```text
WorkflowDefinitionService
WorkflowRunService
InputService
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

Item=`workflow_id/name/version/description/definition_hash/source_kind/source_display`。

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

### 5.1 `wf_start`

Request:

```text
workflow_ref non-empty
inputs object default {}
priority signed64 optional
source_identity non-empty optional
request_id optional
```

Priority resolution:

```text
request priority present -> request value
omitted -> Workflow Definition priority
```

Response:

```text
workflow_run_id
workflow_id
status/conclusion
priority
wait_reason
created_at
```

Run start時`01/08` System baseline/effective settings/Retention/Registry version snapshot固定。

Concurrency scopeは`(workflow_id, resolved group)`。QueueならRun作成、`status=queued, wait_reason=concurrency`。`on-limit=reject`ならRun ID無しで`concurrency_limit_reached`。

### 5.2 `wf_run_list`

Filters=`workflow_id/status/conclusion/created_from/created_to/limit/cursor`。

Order=`created_at DESC,workflow_run_id ASC`。

Item:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
progress nullable
parent_workflow_run_id nullable
root_workflow_run_id
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

Implications:

- attempts -> jobs
- steps -> attempts -> jobs
- artifacts -> jobs

Base Response:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
pause_requested/cancel_requested
parent_workflow_run_id/root_workflow_run_id
source_identity nullable
progress nullable
created_at/started_at/completed_at
output_metadata nullable
jobs nullable
dynamic_groups nullable
events_summary nullable
```

`include_jobs=true`:

- `jobs` = concrete Job Run list
- `dynamic_groups` = Definition上のDynamic template aggregate list

Concrete Job item:

```text
job_run_id/job_key/job_template_key
executor/status/conclusion
priority/progress
current_attempt_id
failure nullable
current_artifacts nullable
attempts nullable
```

Dynamic group item:

```text
template_id
status=queued|running|completed
conclusion nullable|success|failure|cancelled|skipped|blocked
expansion_count
generated_job_count
failure nullable
```

Nested templateもWorkflow Run全体aggregate groupとして1 item。

Attempt item:

```text
attempt_id/attempt_no
status/conclusion
input_digest
output_metadata nullable
failure nullable
started_at/completed_at
log_metadata
steps nullable
artifacts nullable
```

`log_metadata`=`available,size_bytes,updated_at?,deleted_at?`。

`include_steps=true`でStep summary、`include_artifacts=true`かつAttempts含有時はAttempt全generation ArtifactRef/producer metadata。Attempts未指定の`current_artifacts`はcurrent successful Attemptのcurrent named Artifact mapだけ。

`events_summary`:

```text
count
latest_event_type nullable
latest_created_at nullable
```

Authorization後callerが閲覧可能な当該Run owner Eventだけを集計。Retention済みEventはcount対象外。

Input/Output/Log/Event本文無し。

### 5.4 `wf_pause`

Non-terminal root Run。Already paused=same-state success。

### 5.5 `wf_resume`

Paused root Run。New requestでnon-pausedならinvalid_state。Replayはstored result。

Paused concurrency holderはslotを保持しているため通常は同slotでresumeする。`wait_reason=concurrency`のpaused waiterを許可する実装ではresume後もslot取得までqueued待機とする。

### 5.6 `wf_cancel`

Request=`workflow_run_id,reason?,request_id?`。Non-terminal root only。

Cancel/result race=`10`。

### 5.7 `wf_priority_update`

Request=`workflow_run_id,priority,request_id?`。

Root non-terminal Runのみ。1 transactionでroot + 全non-terminal descendant Child Run priorityを同値へ更新。Running Job preempt無し。

Response:

```text
workflow_run_id
priority
updated_descendant_count
status/conclusion
updated_at
```

Child direct update=`child_run_direct_control_forbidden`。

### 5.8 `wf_retry`

Request=`workflow_run_id,job_run_id,request_id?`。Input override field無し。

Response:

```text
workflow_run_id/job_run_id
run_attempt
run_status
run_conclusion nullable
wait_reason nullable
job_status
retry_not_before nullable
updated_at
```

Request時点new Attempt無し。

Non-terminal Run retryはrun_attemptを増やさない。Completed failure Run retryは`10`どおりreopenしてrun_attemptを増やし、Workflow Output current pointerをclearする。

Completed RunのConcurrency再取得で:

- queue -> success response、`run_status=queued, wait_reason=concurrency`
- reject -> `concurrency_limit_reached` 409、Run/Job state変更無し

## 6. Input inspection

Input本文はRun infoへ埋め込まず専用API。

### `wf_input_info`

Exactly one selector:

```text
workflow_run_id xor job_run_id xor attempt_id
```

Resolution:

- workflow_run_id -> Workflow Run input snapshot
- attempt_id -> exact Attempt persistent Input
- job_run_id:
  1. queued + pending snapshotあり -> pending Input
  2. otherwise current_attempt_idあり -> current/latest Attempt Input
  3. otherwise unavailable

Response:

```text
source_type=workflow_run|job_run|attempt
source_id
available
resolved_from=workflow|pending_job|attempt|null
input_digest nullable
has_secret_bindings boolean
size_bytes nullable
```

Workflow InputはSecret binding無し。`size_bytes`=canonical-json-v1 UTF-8 byte size。

### `wf_input_read`

Same selector + optional `select: JMESPath string`。

Response:

```text
source_type/source_id
resolved_from
selected boolean
value any JSON object/value
```

Job/Attempt Inputはpersistent Inputを返し、materialized Secret valueは絶対に返さない。Secret fieldはreference stringのまま。

Unavailable=`input_data_unavailable`。MCP response上限超過=`response_too_large`、silent truncate無し。

## 7. Output

### `wf_output_info`

Exactly one selector=`workflow_run_id xor job_run_id xor attempt_id`。

Response=`source_type/source_id/available/storage_kind/size_bytes/digest`。

Manual Retryでcompleted Runをreopenした直後のWorkflow Run Outputは`available=false`。Past Attempt/Job Outputは履歴selectorで引き続き読める。

### `wf_output_read`

Same selector + optional JMESPath `select`。

Response=`source_type/source_id/selected/value`。

MCP response上限超過=`response_too_large`、silent truncate無し。

## 8. External Task

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

External InputにはSecretが存在しない。

### `wf_task_claim`

Request=`task_id? / workflow_run_id? / job_template_key? / request_id?`。Task ID指定時他filter禁止。No candidate=`{"task":null}`。

Task object=`task_id/lease_id/lease_expires_at/workflow_run_id/job_run_id/attempt_id/job_key/input`。

Candidate ordering=`07`。Task `available_at`を使用しJob `ready_at`を使わない。

ClaimantはActor/client principalからCore生成。

### `wf_task_submit`

Request=`task_id,lease_id,result,artifacts=[],claim_next=false,request_id?`。

Response:

```text
submitted=true
workflow_run_id/job_run_id/attempt_id
job_status/job_conclusion
failure nullable
next_task nullable
```

Validation failureをterminal failureとして受理した場合もsubmitted=true。

`claim_next=true`はsubmit terminal state + optional next Lease + full response/idempotencyを同一DB transaction。Replay時追加claim無し。

`now >= lease_expires_at`は`lease_expired` conflictでresult不採用。

Lease heartbeat/renew/extend/transfer API無し。

## 9. Human Review

### `wf_review_list`

Filters=`workflow_run_id?,status?,limit,cursor`。Order=`created_at ASC,review_id ASC`。

### `wf_review_info`

Request=`review_id`。Input/outcome/comment/actor/timestamps。

### `wf_review_submit`

Request=`review_id,outcome approve|reject,comment?,request_id?`。

Completed/cancelled rewrite不可。同idempotency key replayのみ元response。

## 10. Artifact / Log / Event / Runner

### `wf_artifact_info`

Request=`artifact_id`。Response=canonical public ArtifactRef + producer/deletion metadata。Store path無し。Cross-run read Authorization必須。

Artifact metadata retentionでrow削除済みなら`not_found`。

### `wf_log_read`

Request:

```text
attempt_id
offset_bytes>=0 optional
limit_bytes 1..1048576 default65536
tail_lines 1..10000 optional
```

Tailとoffset同時禁止。

Response=`content,next_offset_bytes?,truncated,size_bytes,updated_at?`。

Deleted log=`log_data_unavailable`。

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

複数owner filter指定時は同じresource chainであることを検証。不一致=`validation_failed`。

No owner filterでもAuthorizationProvider filtered scope必須。

Order=`created_at DESC,event_id DESC`。

Item=`event_id/type/version/owner IDs/runner_id/actor/source/request_id/payload?/created_at`。

### `wf_runner_info`

Request=`pool?`。Response=`pools[{name,configured_count,runners[]}]`。

## 11. 禁止generic mutation

無し:

```text
wf_job_mark_success
wf_job_override_conclusion
wf_job_skip
wf_job_force_complete
wf_review_rewrite
```

Failed Jobは原因修正後Retry。Skip/許容failureはDefinitionで事前定義。

## 12. MCP namespace / canonical tools

`system_namespace` non-empty、推奨`^[a-z][a-z0-9_]*$`。Tool collision registration時reject。

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
<ns>_wf_input_info
<ns>_wf_input_read
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

## 13. HTTP Adapter v1

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
GET  /workflow-runs/{workflow_run_id}/input-info
GET  /workflow-runs/{workflow_run_id}/input
GET  /jobs/{job_run_id}/input-info
GET  /jobs/{job_run_id}/input
GET  /attempts/{attempt_id}/input-info
GET  /attempts/{attempt_id}/input
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

## 14. HTTP status

```text
200 read / successful non-create mutation
201 top-level Workflow Run creation
400 request/definition/input/expression/domain validation
401 unauthenticated
403 forbidden
404 not_found
409 state/idempotency/lease/concurrency conflict
413 explicit Adapter size policy
500 internal_error
```

422無し。Idempotency replayはoriginal successful status/body。

## 15. Error model

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
input_data_unavailable
retry_input_unavailable
action_version_mismatch
validator_version_mismatch
action_contract_error
validator_contract_error
idempotency_conflict
concurrency_limit_reached
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

## 16. Idempotency

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
4. optional fast lookup
5. domain/expensive validation + temp prepare
6. `BEGIN IMMEDIATE`
7. key/hash再確認
8. current domain state再確認
9. side effect + full result + completed idempotency row commit
10. Adapter response

DB `reserved` row無し。Concurrent same-keyはSQLite write serialization + commit recheck。

TTL内same hash=replay、different hash=conflict。`now >= expires_at`でexpired、replace可。

## 17. Authorization / pagination

全public operation authorize。Input readも通常resource read権限を要求。List/Eventはfiltered scope。Cursor opaque。Limit1..200 default50。

## 18. Python API

Canonical Service request/response modelを直接使用。Adapter独自意味変更禁止。

## 19. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 20. 受入条件

1. Definition/Run separation
2. canonical timestamp exact format
3. request unknown field/Actor injection reject
4. root start priority default/override
5. concurrency start queue/reject code
6. priority update descendant propagation/Child direct reject
7. Run info concrete jobs + Dynamic group queued/running/completed
8. include attempts/steps/artifacts implications
9. events_summary exact shape
10. Input info/read selector/current pending resolution
11. Input read never materializes Secret
12. Output selector/read + reopen output unavailable
13. wf_retry nonterminal/completed run_attempt semantics
14. wf_retry concurrency reacquire queue/reject response
15. task claim no-candidate null + available_at ordering delegation
16. expired Lease submit reject
17. submit+claim_next same transaction/replay
18. no lease renew APIs
19. Human rewrite/manual Job mutation無し
20. Artifact metadata retention -> not_found
21. Artifact/Log errors
22. Event filters/order/resource-chain validation
23. namespaced MCP collision
24. HTTP exact routes/Idempotency-Key/status/no422
25. idempotency canonical hash/no-reserved/commit recheck/expiry
26. all read/write Authorization
