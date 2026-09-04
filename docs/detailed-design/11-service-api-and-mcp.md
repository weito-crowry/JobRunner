# 11. Service API / MCP / HTTP 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 基本原則

1. Web/MCP/Python APIは同じService layer。
2. AdapterはDB/Storeを直接更新しない。
3. Public read/writeはAuthorizationProviderを通す。
4. State-changing operationはoptional `request_id`。
5. MCP public tool名は親namespace必須。
6. Large Log/Output本文をRun infoへ埋め込まない。
7. **Workflow Definition と Workflow Run をAPI名で明確に分ける。**
8. HTTP Adapterのroute contractは本書で固定し、WebUI画面設計へ委譲しない。

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

Canonical logical names:

```text
wf_definition_list
wf_definition_info
```

### `wf_definition_list`

親Workflow Resolver/Registryから現在利用可能なDefinitionを列挙するread-only operation。

最低限返す:

```text
workflow_id
name
version
description optional
definition_hash
source_kind
source_display optional
```

Raw source YAML全体は既定responseへ含めない。

### `wf_definition_info`

Canonical workflow IDまたは解決可能referenceを指定し、Definition metadataとInput/Output schema、Job summaryを返す。

Option:

```text
include_source_yaml=false default
include_jobs=true default
```

Secret valueはDefinitionに存在しない。

## 5. Workflow Run operations

Canonical logical names:

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

`wf_list` / `wf_info` という曖昧aliasはMVP canonical APIとして提供しない。

### `wf_start`

Request:

```text
workflow reference
inputs
priority optional
source_identity optional
request_id optional
```

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

Output body/Execution Log本文は含めない。Inlineで小さくても同じcontract。

### `wf_retry`

`workflow_run_id + job_run_id`。

Eligibilityは`10-retry-recovery-cancel.md`をSource of Truthとし、少なくとも:

- target Job conclusion=`failure`
- latest failed Attemptが存在
- persistent Input Snapshotが存在
- success/cancelled Runではない

を要求する。

Attempt/Input Snapshot無しは:

```text
retry_input_unavailable
```

でsame Run retryを拒否する。Input変更不可。Completed/failure Runはexplicit reopen。

Child Run direct controlは禁止。

## 6. Output operations

```text
wf_output_info
wf_output_read
```

Sourceはexactly one:

- Workflow Run current Workflow Output
- Job Run current successful Output
- specific Attempt Output

`wf_output_read` はPayloadStoreを透過loadして元JSON valueを返す。

### `select`

Optional JMESPath expression。

- full Output load + integrity verify後に評価
- returned valueもJSON-compatible
- select無しはfull Output
- MCP response上限超過時はsilent truncateせず `response_too_large`
- Python/HTTPはtransport能力に応じstreaming可能だがJSON意味を変えない

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

`wf_artifact_info`はmetadata/history。Managed Artifact実体downloadをMVP public Service必須にはしない。

Log readはattempt_id + offset/tail。External path不可。

## 10. MCP namespace / canonical tools

Core logical nameの先頭に親namespaceを付ける。

例:

```text
Core: wf_run_info
Novel: novel_wf_run_info
FX: fx_wf_run_info
```

`system_namespace`必須。親はtool subsetを非公開可能。

MVP canonical public tool:

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

親システムがmount pathを変更できるが、JobRunner内の標準prefixは:

```text
/api/jobrunner/v1
```

Canonical routes:

```text
GET  /api/jobrunner/v1/workflow-definitions
GET  /api/jobrunner/v1/workflow-definitions/{workflow_id}

POST /api/jobrunner/v1/workflow-runs
GET  /api/jobrunner/v1/workflow-runs
GET  /api/jobrunner/v1/workflow-runs/{workflow_run_id}
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/pause
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/resume
POST /api/jobrunner/v1/workflow-runs/{workflow_run_id}/cancel
PATCH /api/jobrunner/v1/workflow-runs/{workflow_run_id}       # priority only in MVP
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

Path IDはURL percent-encodingする。`workflow_id`のcanonical stringをpath segmentとして扱う。

### Query mapping

```text
workflow-runs list -> status/conclusion/workflow_id/created_from/created_to/limit/cursor
reviews list -> workflow_run_id/status/limit/cursor
output -> select optional
log -> offset/limit/tail_lines
```

### Idempotency header

HTTP state-changing requestはoptional:

```text
Idempotency-Key: <opaque client key>
```

をService `request_id`へmappingする。HTTP bodyに別の`request_id` fieldを重複して持たせない。

## 12. HTTP status mapping

```text
200  read / idempotent replay / successful state change
201  workflow start等でnew resource作成
400  request/definition/input/expression validation
401  parent authenticationでunauthenticated
403  authorization forbidden
404  not_found policy時
409  conflict/invalid_state/idempotency_conflict/lease conflict
413  HTTP Adapterが明示response/request size policyで拒否する場合
422  domain valueは構文上validだがcontract不適合の場合に使用可。MVPでは400へ統一してもよい
500  internal_error
```

**MVP canonical mappingは validation/domain contract errorを400へ統一する。** 422は将来拡張用で、Adapterごとに勝手に使い分けない。

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

Scopeはsystem namespace + resource + AccessScope + Actor/client principal。

TTL内same hash replay / different hash conflict。TTL後はexpired rowをtransactional replacementしてsame keyをnew requestとして利用可能。

## 15. Validation order

State change:

1. request schema
2. Actor/AccessScope
3. Authorization
4. idempotency lookup/reservation
5. domain validation
6. side effect + Event + result transaction
7. response

Secret値をrequest hash/Eventへ保存しない。

## 16. Read authorization / pagination

Definition list/info、Run list/info、output、task、review、artifact、log、runnerを全てauthorize。

ListはProviderのfiltered scopeをqueryへ適用。

List/Event/Artifact historyはopaque cursor pagination。

## 17. Python API

Python APIも同じcanonical logical operationを呼ぶ。Adapter独自の意味変更を禁止する。

## 18. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 19. 受入条件

1. Definition list/info と Run list/infoが別operation
2. ambiguous `wf_list/wf_info` canonical alias無し
3. namespaced MCP canonical names
4. all read/write Authorization
5. Run info Output body無し
6. output info/read inline/blob透過
7. output JMESPath select
8. MCP large response fail-closed/no truncate
9. retry_input_unavailable Service mapping
10. HTTP v1 exact routes/methods
11. HTTP Idempotency-Key mapping
12. HTTP error status mapping
13. task/review operations
14. child direct control reject
15. idempotency scope/TTL
16. cursor pagination
