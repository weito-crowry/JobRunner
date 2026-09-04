# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v1.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `07`, `08`, `09`, `10`, `12`

## 1. 基本原則

1. MCP/HTTP/Pythonは同じService layer。
2. AdapterはDB/Store直接更新禁止。
3. Public read/writeはAuthorizationProvider必須。
4. State-changing operationはoptional `request_id`。
5. MCP toolは親namespace必須。
6. Large Input/Output/State/Log本文をRun infoへ埋め込まない。
7. Workflow DefinitionとWorkflow RunをAPI名で分離。
8. Canonical request/response modelを本書で固定。
9. Generic Job state override API無し。
10. Core Service modelはstrict/no-coercion。
11. External Lease claimant identityはrequest bodyではなくcurrent ActorContext/AccessScopeからCoreが算出する。

## 2. 共通model / Adapter parsing

Pydantic typed model + JSON-compatible serialization。`01` strict type semanticsを使う。

- Core ID=opaque string
- ActorContext/AccessScopeはAdapter/parentから別argument注入
- canonical `actor_principal_key`=`12 §2.1`
- Python/MCP `request_id` optional
- HTTPは`Idempotency-Key` header -> request_id
- Unknown request field reject
- Timestamp=`08` canonical UTC `YYYY-MM-DDTHH:MM:SS.ffffffZ`

External `wf_task_claim/wf_task_submit` はActorContextにnon-empty `actor_id`必須。欠落=`claimant_identity_required`。`claimant_key`をpublic request fieldとして受け取らない。

### 2.1 Core model

Core Service modelは文字列からinteger/boolean等へcoerceしない。

```text
"50" != 50
"true" != true
true != 1
```

Python/MCPはtyped JSON値をそのままCore modelへ渡す。

### 2.2 HTTP query/path parser

HTTPはtransport上query/pathが文字列なのでAdapterだけがschemaに従って明示parseする。

- integer query: ASCII base-10、fieldのsigned/nonnegative rangeを検証。float/exponent/whitespace coercion無し
- boolean query: exact lowercase `true|false` のみ
- timestamp query: canonical UTC形式へparse/normalize後Coreへ渡す。invalid -> 400
- enum/string:文字列のまま、trim/lowercaseを勝手にしない
- JSON bodyはJSON parserが作った型をCoreへ渡し、`"1"`をinteger fieldへ変換しない

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
WorkflowStateService
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

Request=`workflow_ref,include_source_yaml=false,include_jobs=true`。

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

`wf_definition_list/info`はbrowse cacheを利用可能だが、`wf_start`の実行開始source readは`01`どおり必ずcurrent source bytesを再取得する。

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

Priority=request value if present, otherwise Workflow Definition priority。

Response:

```text
workflow_run_id
workflow_id
status
conclusion nullable
priority
wait_reason nullable
concurrency_queued_at nullable
created_at
```

Run start時`01/08` snapshot固定。Concurrency scope=`(workflow_id,group)`。

- slot admission success/no concurrency -> Run作成、`status=running, wait_reason=null, concurrency_queued_at=null`
- queue -> Run作成、`status=queued, wait_reason=concurrency, concurrency_queued_at=<queue time>`
- reject -> Run ID無し `concurrency_limit_reached`

MVPのRun `queued`はConcurrency待ち専用。

### 5.2 `wf_run_list`

Filters=`workflow_id/status/conclusion/created_from/created_to/limit/cursor`。

Order=`created_at DESC,workflow_run_id ASC`。

Item:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
concurrency_queued_at nullable
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

Implications: attempts -> jobs、steps -> attempts、artifacts -> jobs。

Base Response:

```text
workflow_run_id/workflow_id/workflow_version
status/conclusion/priority/run_attempt/wait_reason
concurrency_queued_at nullable
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

Concrete Job item:

```text
job_run_id/job_key/job_template_key
executor/status/conclusion
priority/progress
current_attempt_id nullable
current_task_id nullable
current_review_id nullable
current_child_workflow_run_id nullable
failure nullable
current_artifacts nullable
attempts nullable
```

Navigation resolution:

- current_task_id = current Attemptに属するExternal Task ID
- current_review_id = current Attemptに属するHuman Review ID
- current_child_workflow_run_id = current Attemptから作られたChild Run ID
- executor不一致/未作成/terminal history onlyならnullable

Dynamic group item:

```text
template_id
status=queued|running|completed
conclusion nullable|success|failure|cancelled|skipped|blocked
expansion_count
generated_job_count
failure nullable
```

Attempt item:

```text
attempt_id/attempt_no
status/conclusion
input_digest
output_metadata nullable
failure nullable
started_at/completed_at
log_metadata
external_task_id nullable
review_id nullable
child_workflow_run_id nullable
steps nullable
artifacts nullable
```

Step summary (`include_steps=true`):

```text
step_id
sequence
name
status/conclusion
start_metadata nullable
finish_metadata nullable
started_at/completed_at
```

MVPでは同一Attemptに同時open Step最大1。

Attempt navigation IDはそのAttemptに属するexact rowを返す。Concurrency rejectでReusable Child未作成なら`child_workflow_run_id=null`。

`log_metadata`=`available,size_bytes,updated_at?,deleted_at?`。

`events_summary`=`count,latest_event_type?,latest_created_at?`。

Input/Output/State/Log/Event本文無し。

### 5.4 `wf_pause`

Root non-terminal Runのみ。

- `status=running` -> `status=paused, wait_reason=null, concurrency_queued_at=null`。Concurrency設定時はslot保持
- `status=queued, wait_reason=concurrency` -> `status=paused, wait_reason=concurrency`。元`concurrency_queued_at`保持、slot無し、wake候補外
- already paused -> same-state success

Child direct pause=`child_run_direct_control_forbidden`。

### 5.5 `wf_resume`

Paused root Runのみ。New requestでnon-pausedなら`invalid_state`。Replayはstored result。

- paused + `wait_reason=null` -> `status=running, concurrency_queued_at=null`。admitted holderは同じslotで再開
- paused + `wait_reason=concurrency` -> `status=queued, wait_reason=concurrency`。元`concurrency_queued_at`を保持してslot取得まで待機

### 5.6 `wf_cancel`

Request=`workflow_run_id,reason?,request_id?`。Non-terminal root only。Race=`10`。

### 5.7 `wf_priority_update`

Request=`workflow_run_id,priority,request_id?`。

Root non-terminal only。root + all non-terminal descendant Child Runへ同値伝播。Preempt無し。

Response=`workflow_run_id,priority,updated_descendant_count,status,conclusion,updated_at`。

HTTP `PATCH /workflow-runs/{id}` bodyはexact `{ "priority": <signed64 integer> }`。Unknown body field reject。

### 5.8 `wf_retry`

Request=`workflow_run_id,job_run_id,request_id?`。Input override無し。

Response:

```text
workflow_run_id/job_run_id
run_attempt
run_status
run_conclusion nullable
wait_reason nullable
concurrency_queued_at nullable
job_status
retry_not_before nullable
updated_at
```

Non-terminal retryはrun_attempt不変。Completed failure reopenは`10`。

Concurrency reacquire:

- admitted -> Run running / queue timestamp null
- queue -> success response、Run queued/concurrency + fresh `concurrency_queued_at`
- reject -> `concurrency_limit_reached` 409、state変更無し

Retry pendingのtarget Jobは`status=queued, conclusion=null, completed_at=null`。

## 6. Input inspection

### `wf_input_info`

Exactly one selector=`workflow_run_id xor job_run_id xor attempt_id`。

Job resolution: pending snapshot -> current/latest Attempt -> unavailable。

Response:

```text
source_type/source_id
available
resolved_from=workflow|pending_job|attempt|null
input_digest nullable
has_secret_bindings boolean
size_bytes nullable
```

### `wf_input_read`

Same selector + optional JMESPath `select`。

Job/Attemptはpersistent Inputを返し、Secret valueはreference stringのまま。Unavailable=`input_data_unavailable`。

## 7. Output

### `wf_output_info`

Exactly one selector=`workflow_run_id xor job_run_id xor attempt_id`。

Response=`source_type/source_id/available/storage_kind/size_bytes/digest`。

Reopened Run直後はWorkflow Output `available=false`。Past Attempt/Job Outputは読める。

### `wf_output_read`

Same selector + optional JMESPath `select`。MCP oversized=`response_too_large`、silent truncate無し。

## 8. Workflow State read / history

Public mutation APIはMVPに持たない。State writeはinternal Action Runtime Handleだけ。

全操作は`workflow_state.read` Authorization。

### 8.1 `wf_state_list`

Request=`workflow_run_id,limit,cursor`。Order=`name ASC`。

Item=`name,revision,size_bytes,updated_at`。

`size_bytes`=current `value_json` canonical UTF-8 bytes。

### 8.2 `wf_state_read`

Request=`workflow_run_id,name non-empty`。

Response=`workflow_run_id,name,revision,value,size_bytes,updated_at`。Missing=`not_found`。

### 8.3 `wf_state_history`

Request=`workflow_run_id,name?,include_values=false,limit,cursor`。

Order=`created_at DESC,history_id DESC`。

Item:

```text
history_id
name
revision
job_run_id nullable
attempt_id nullable
step_id nullable
actor nullable
created_at
old_value optional
new_value optional
```

`include_values=false`ではold/newを含めない。`true`では保存済みJSON値を返す。

Large State/history valueはRun info/Listへ埋めず、read/historyのみ。Adapter/MCP response上限超過=`response_too_large`、silent truncate無し。

State historyはAttempt failure後も残り得る。

## 9. External Task

### 9.1 `wf_task_info`

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

`current_lease`:

```text
status
expires_at nullable
claimed_by_self boolean
lease_id nullable
```

Current caller principal=`12 actor_principal_key`。

- active Lease `claimant_key == current actor_principal_key` の場合のみ`claimed_by_self=true`かつ`lease_id`を返す
- `actor_id`が無いcallerは`claimed_by_self=false`としてLease IDを復元しない
- 他principal claimなら`lease_id=null`。claimant identity/lease tokenを一般readへ露出しない
- inactive/無しなら必要最小metadataだけ
- `claim_history_summary`へlease_idやclaimant/capability情報を含めない

Task submitはLease IDだけでなくcurrent Actor principal ownershipも再確認するため、他Actorがtokenだけでsubmitできない。

### 9.2 `wf_task_claim`

Request=`task_id? / workflow_run_id? / job_template_key? / request_id?`。Task ID指定時他filter禁止。No candidate=`{"task":null}`。

Current ActorContext `actor_id` non-empty必須。欠落=`claimant_identity_required`。

Task object=`task_id/lease_id/lease_expires_at/workflow_run_id/job_run_id/attempt_id/job_key/input`。

Claim成功時Lease `claimant_key=current actor_principal_key`。Ordering=`07`、Task `available_at`使用。Stable `job_key`はopaque IDsより先にtie-break。

### 9.3 `wf_task_submit`

Request=`task_id,lease_id,result,artifacts=[],claim_next=false,request_id?`。

Current ActorContext `actor_id` non-empty必須。`claimant_key`はrequest bodyに含めない。

Response=`submitted=true,workflow_run_id,job_run_id,attempt_id,job_status,job_conclusion,failure?,next_task?`。

`claim_next=true`はsubmit + optional next Lease + idempotencyをsame transaction。`now >= lease_expires_at`はresult不採用。

Transaction内でtask/lease current ownership、Lease ID、`claimant_key == current actor_principal_key`、Actor/Scope Authorizationを再確認する。`claim_next`で取得する新Leaseもsame actor_principal_keyを使う。

Lease heartbeat/renew/extend/transfer無し。

## 10. Human Review

`wf_review_list`: filters=`workflow_run_id?,status?,limit,cursor`, order created_at ASC/id ASC。

`wf_review_info`: review_id -> Input/outcome/comment/actor/timestamps。

`wf_review_submit`: review_id,outcome approve|reject,comment?,request_id?。Rewrite不可。

## 11. Artifact / Log / Event / Runner

### `wf_artifact_info`

artifact_id -> public ArtifactRef + producer/deletion metadata。Store path無し。Metadata retention後=`not_found`。

### `wf_log_read`

Request:

```text
attempt_id
offset_bytes >=0 optional
limit_bytes 1..1048576 default65536
tail_lines 1..10000 optional
```

Tail/offset同時禁止。Deleted=`log_data_unavailable`。

Execution Log fileはUTF-8 text。`offset_bytes`は0または過去responseの`next_offset_bytes`利用を正規利用とする。任意offsetがUTF-8 code point境界でない場合は`invalid_log_offset` 400とし、暗黙に前後へ丸めない。

Response=`content,next_offset_bytes?,truncated,size_bytes,updated_at?`。

### `wf_event_list`

Request=`workflow_run_id?,job_run_id?,attempt_id?,types?,created_from?,created_to?,include_payload=true,limit,cursor`。

複数owner filterはsame resource chain必須。Order=`created_at DESC,event_id DESC`。

### `wf_runner_info`

Request=`pool?`。Response=`pools[{name,configured_count,runners[]}]`。

## 12. 禁止generic mutation

無し:

```text
wf_job_mark_success
wf_job_override_conclusion
wf_job_skip
wf_job_force_complete
wf_review_rewrite
wf_state_set
wf_state_delete
```

State mutationはtrusted internal Action Runtime Handleのみ。

## 13. MCP namespace / canonical tools

`system_namespace` non-empty、推奨`^[a-z][a-z0-9_]*$`。Collision reject。

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
<ns>_wf_state_list
<ns>_wf_state_read
<ns>_wf_state_history
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

Actor/AccessScopeはtool inputへ露出しない。External claim/submitのclaimant identityもAdapter/parent injectionから算出しtool inputへ露出しない。

## 14. HTTP Adapter v1

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
GET  /workflow-runs/{workflow_run_id}/state
GET  /workflow-runs/{workflow_run_id}/state/read?name=<encoded>
GET  /workflow-runs/{workflow_run_id}/state-history
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

`state-history` query supports `name`, `include_values`, pagination。

Opaque Core IDsのみpath parameter。Workflow ref/state name等はquery/bodyで安全にURL encode。

`PATCH /workflow-runs/{id}`はpriority update専用でbody exact `{priority}`。

State-changing HTTP optional `Idempotency-Key` header。Body request_id無し。

## 15. HTTP status

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

## 16. Error model

```text
code/message/retryable/field-or-path?/details?
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
claimant_identity_required
lease_conflict/lease_expired
runner_unavailable
payload_missing/payload_digest_mismatch
artifact_access_forbidden/artifact_data_unavailable
log_data_unavailable
invalid_log_offset
secret_binding_invalid
successful_job_result_not_reusable
dynamic_expansion_not_reusable
response_too_large
child_run_direct_control_forbidden
internal_error
```

## 17. Idempotency

対象=`wf_start/wf_pause/wf_resume/wf_cancel/wf_retry/wf_priority_update/wf_task_claim/wf_task_submit/wf_review_submit`。

Identity=`scope + operation + request_id`。Request hash=`08` canonical Service request excluding request_id/transport fields。

Canonical actor isolationは`12 §2.1 actor_principal_key`。`08`どおりprincipal keyにAccessScopeを含むためIdempotency scopeへAccessScopeを別途二重追加しない。

Flow=Schema -> Actor/Scope -> Auth -> optional fast lookup -> prepare -> `BEGIN IMMEDIATE` -> key/hash再確認 -> domain state再確認 -> side effect+full result row commit -> response。

No reserved row。TTL内same hash replay/different conflict。`now >= expires_at`でreplace可。

## 18. Authorization / pagination

全public operation authorize。State list/read/history=`workflow_state.read`。

Task info/claim/submitはExternalTask resource Authorizationに加え:

- claim/submitはnon-empty actor_id必須
- submitはcurrent actor_principal_keyとLease claimant_key exact一致必須
- task infoのactive lease_idは同principalのcallerだけ取得可能

List/Eventはfiltered scope。Limit1..200 default50。

## 19. Python API

Canonical strict Service request/response modelを直接使用。Adapter独自意味変更禁止。

## 20. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 21. 受入条件

1. strict/no-coercion Core Service models
2. HTTP explicit integer/boolean/timestamp parsing
3. Definition/Run separation + start fresh source path
4. admitted start=running / concurrency queue=queued + queue timestamp
5. Run list/info/start/retry expose concurrency_queued_at consistently
6. pause/resume admitted holder vs concurrency waiter preserves queue timestamp
7. root priority/HTTP PATCH exact body
8. Run info concrete Jobs/Dynamic groups/Step exact shapes
9. Job/Attempt task-review-child navigation IDs
10. Input info/read Secret reference only
11. Output read/reopen unavailable
12. State list metadata-only/current read/history values optional
13. State history persists failed Attempt writes
14. no public State mutation
15. wf_retry Job terminal reset/run_attempt/concurrency response
16. task claim/submit actor_id required
17. canonical actor_principal_key used for claimant/idempotency
18. task available_at ordering + job_key tie-break
19. task_info does not expose another claimant's lease_id
20. self claimant can recover active lease_id
21. submit verifies lease + claimant ownership
22. submit+claim_next same claimant transaction/replay
23. Human mutation restrictions
24. Artifact/Log byte-offset/Event contracts
25. namespaced MCP includes State tools
26. HTTP exact State routes
27. status/no422
28. idempotency commit recheck/expiry
29. all read/write Authorization
