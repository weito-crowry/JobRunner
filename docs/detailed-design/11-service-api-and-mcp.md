# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v0.6
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 基本原則

1. MCP/HTTP/Python APIは同じService layer。
2. AdapterはDB/Storeを直接更新しない。
3. Public read/writeはAuthorizationProviderを通す。
4. State-changing operationはoptional `request_id`。
5. MCP public tool名は親namespace必須。
6. Large Log/Output本文をRun infoへ埋め込まない。
7. Workflow Definition と Workflow Run をAPI名で明確に分ける。
8. HTTP route contractは本書で固定し、WebUI画面設計へ委譲しない。

## 2. Service

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

## 3. Actor / AccessScope

各operationはActorContext/AccessScopeを受ける。

State-changing public callでactor_idが無ければAdapterがclient/session principalを補い、anonymous global principalでidempotency keyを共有しない。

## 4. Workflow Definition operations

```text
wf_definition_list
wf_definition_info
```

### `wf_definition_list`

現在利用可能なDefinitionを列挙するread-only operation。

最低限:

```text
workflow_id
name
version
description optional
definition_hash
source_kind
source_display optional
```

Raw YAML全体は既定responseへ含めない。

### `wf_definition_info`

Canonical `workflow_id`またはresolverが受理するreferenceをrequest fieldとして受け、Definition metadata、Input/Output schema、Job summaryを返す。

```text
workflow_ref
include_source_yaml=false
include_jobs=true
```

HTTPではworkflow ID/referenceをpath segmentへ埋め込まずquery parameterで渡す。Filesystem/Registry由来のcanonical IDに`/`等が含まれてもrouting semanticsと衝突させないためである。

## 5. Workflow Run operations

```text
wf_start
wf_run_list
wf_run_info
wf_pause
wf_resume
wf_cancel
wf_retry
wf_priority_update
```

曖昧な`wf_list/wf_info` canonical aliasはMVPで提供しない。

### `wf_start`

Request:

```text
workflow_ref
inputs
priority optional
source_identity optional
request_id optional
```

`source_identity`はpresentならnon-empty string。Git SHA/package version/build ID等のopaque parent metadataで、Coreは内容解釈しない。

Validation failureではRunを作らない。

### `wf_run_list`

Filter:

```text
workflow_id optional
status optional
conclusion optional
created_from/to optional
limit/cursor
```

### `wf_run_info`

Options:

```text
include_jobs
include_attempts
include_steps
include_artifacts
include_events_summary
include_output_metadata
```

Output metadata:

```text
available
storage_kind
size_bytes
digest
```

Output body/Execution Log本文は含めない。

### `wf_retry`

`workflow_run_id + job_run_id`。Eligibilityは`10`。

Attempt/Input Snapshot無し -> `retry_input_unavailable`。Input変更不可。Child direct control禁止。

## 6. Output operations

```text
wf_output_info
wf_output_read
```

Source exactly one:

- Workflow Run current Workflow Output
- Job Run current successful Output
- specific Attempt Output

`wf_output_read`はPayloadStoreを透過load。

Optional `select` はJMESPath。

- full Output integrity verify後評価
- select無しfull Output
- MCP response上限超過時silent truncateせず `response_too_large`
- Python/HTTPはstreaming可能だがJSON意味を変えない

## 7. External Task operations

```text
wf_task_info
wf_task_claim
wf_task_submit
```

Claim orderingは`07`。`claim_next`も同じ。

## 8. Human Review operations

```text
wf_review_list
wf_review_info
wf_review_submit
```

Submit first-wins。

## 9. Artifact / Log / Runner operations

```text
wf_artifact_info
wf_log_read
wf_runner_info
```

Artifact infoはmetadata/history。Managed実体downloadはMVP public Service必須ではない。

Log readはattempt_id + offset/tail。External path不可。

## 10. MCP namespace / canonical tools

Core logical nameの先頭に親namespace。

```text
Core: wf_run_info
Novel: novel_wf_run_info
FX: fx_wf_run_info
```

`system_namespace`はnon-empty、推奨正規表現:

```text
^[a-z][a-z0-9_]*$
```

Adapter登録時に同一MCP server内tool name collisionを検出してrejectする。

MVP canonical public tools:

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

## 11. HTTP Adapter v1

Standard prefix:

```text
/api/jobrunner/v1
```

親システムはこのrouter全体を別mount prefixの下へ載せられるが、JobRunner内部route suffixを変更しない。

Canonical routes:

```text
GET  /api/jobrunner/v1/workflow-definitions
GET  /api/jobrunner/v1/workflow-definitions/info?workflow_ref=<encoded>

POST /api/jobrunner/v1/workflow-runs
GET  /api/jobrunner/v1/workflow-runs
GET  /api/jobrunner/v1/workflow-runs/{workflow_run_id}
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/pause
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/resume
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/cancel
PATCH /api/jobrunner/v1/workflow-runs/{workflow_run_id}
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/jobs/{job_run_id}/retry

GET  /api/jobrunner/v1/workflow-runs/{workflow_run_id}/output-info
GET  /api/jobrunner/v1/workflow-runs/{workflow_run_id}/output
GET  /api/jobrunner/v1/jobs/{job_run_id}/output-info
GET  /api/jobrunner/v1/jobs/{job_run_id}/output
GET  /api/jobrunner/v1/attempts/{attempt_id}/output-info
GET  /api/jobrunner/v1/attempts/{attempt_id}/output

GET  /api/jobrunner/v1/external-tasks/{task_id}
POST /api/jobrunner/v1/external-tasks/claim
POST /api/jobrunner/v1/external-tasks/{task_id}/submit

GET  /api/jobrunner/v1/reviews
GET  /api/jobrunner/v1/reviews/{review_id}
POST /api/jobrunner/v1/reviews/{review_id}/submit

GET  /api/jobrunner/v1/artifacts/{artifact_id}
GET  /api/jobrunner/v1/attempts/{attempt_id}/log
GET  /api/jobrunner/v1/runners
```

Opaque generated IDs (`wr_`, `jr_`, etc.)だけをpath parameterに使う。Workflow referenceはquery/body field。

### Query mapping

```text
workflow-definitions/info -> workflow_ref, include_source_yaml, include_jobs
workflow-runs list -> status/conclusion/workflow_id/created_from/created_to/limit/cursor
reviews list -> workflow_run_id/status/limit/cursor
output -> select optional
log -> offset/limit/tail_lines
```

### Idempotency header

State-changing HTTP requestはoptional:

```text
Idempotency-Key: <opaque client key>
```

をService `request_id`へmapping。Bodyに別`request_id`を持たせない。

## 12. HTTP status mapping

MVP canonical mapping:

```text
200  read / successful non-create state change
201  successful new top-level Workflow Run creation
400  request/definition/input/expression/domain contract validation
401  parent authenticationでunauthenticated
403  authorization forbidden
404  not_found policy時
409  state conflict / idempotency conflict / lease ownership conflict
413  Adapterの明示request/response size policyで拒否
500  internal_error
```

MVPでは422を使用しない。

### Idempotency replay status

Idempotency replayは**最初の成功responseのHTTP status + bodyをそのまま再生**する。

例:

- 初回`wf_start`=201 -> replayも201
- 初回pause=200 -> replayも200

そのため「replayは常に200」という特別規則は設けない。Persistenceのidempotency resultはAdapterがoriginal statusを復元できるmetadataを保持する。

`response_too_large` はMCP固有のtool errorが基本。HTTPで同様のAdapter上限を設けて拒否する場合は413。

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
wf_start
wf_pause
wf_resume
wf_cancel
wf_retry
wf_priority_update
wf_task_claim
wf_task_submit
wf_review_submit
```

Identity:

```text
scope + operation + request_id
```

Scope=system namespace + resource + AccessScope + Actor/client principal。

TTL内same hash replay / different hash conflict。TTL後expired row transactional replacement可。

Persisted resultにはService response bodyに加えAdapter replay用original HTTP status metadataを持てる。MCP/Pythonは同じService resultを各Adapter形式へ再構成する。

## 15. Validation order

1. request schema
2. Actor/AccessScope
3. Authorization
4. idempotency lookup/reservation
5. domain validation
6. side effect + Event + result transaction
7. response

Secret値をrequest hash/Eventへ保存しない。

## 16. Read authorization / pagination

Definition list/info、Run list/info、output、task、review、artifact、log、runner全てauthorize。

Listはfiltered scope適用。

List/Event/Artifact historyはopaque cursor pagination。

## 17. Python API

Python APIもcanonical logical operation。Adapter独自の意味変更禁止。

## 18. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 19. 受入条件

1. Definition/Run operation separation
2. no wf_list/wf_info alias
3. namespaced MCP collision reject
4. all read/write Authorization
5. Run info body separation
6. output info/read/select
7. MCP large response no truncate
8. retry_input_unavailable mapping
9. HTTP Definition info query route supports slash-containing ID
10. HTTP exact routes/methods
11. Idempotency-Key mapping
12. replay preserves original 201/200 status
13. no HTTP 422 in MVP
14. task/review operations
15. child direct control reject
16. idempotency scope/TTL
17. cursor pagination
