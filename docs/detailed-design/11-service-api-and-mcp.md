# 11. Service API / MCP 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 基本原則

1. Web/MCP/Python APIは同じService layer。
2. AdapterはDB/Storeを直接更新しない。
3. Public read/writeはAuthorizationProviderを通す。
4. State-changing operationはoptional `request_id`。
5. MCP public tool名は親namespace必須。
6. Large Log/Output本文を`wf_info`へ埋め込まない。

## 2. Service

```text
WorkflowService
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

## 4. Workflow operations

```text
wf_start
wf_list
wf_info
wf_pause
wf_resume
wf_cancel
wf_retry
wf_priority_update
```

### `wf_start`

```text
workflow reference
inputs
priority optional
source_identity optional
request_id optional
```

Validation failureではRunを作らない。

### `wf_retry`

`workflow_run_id + job_run_id`。Failed Jobのみ。

Completed/failure Runはexplicit reopen。Success/cancelled Run reject。Input変更不可。

Child Run direct controlは禁止。

## 5. `wf_info`

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

**Output bodyは含めない。** Inlineで小さくても同じcontractに統一する。

Execution Log本文も含めない。

## 6. Output operations

```text
wf_output_info
wf_output_read
```

`wf_output_info`:

- `workflow_run_id` のWorkflow Output、または
- `job_run_id` / `attempt_id` のJob Output

のmetadataを返す。

`wf_output_read` はPayloadStoreを透過的にloadし元のJSON valueを返す。

Request:

```text
workflow_run_id optional
job_run_id optional
attempt_id optional
select optional
```

対象はexactly one sourceへ解決する。

### 6.1 `select`

Large OutputをMCPへ全量返さなくて済むようoptional JMESPath expressionを許可する。

```text
select = "candidates[?score > `0.8`]"
```

- Output全体をload/integrity検証後、JMESPath評価
- returned valueもJSON-compatible
- `select`無しはfull Output
- MCP Adapterはtransport/model tool-result上限を超えるfull responseを検知した場合、silent truncateせず `response_too_large` を返し、`select`利用を案内できる
- Python/HTTP Adapterは各transport能力に応じstreaming/response policyを実装してよいが、値の意味を変えない

## 7. External Task

```text
wf_task_info
wf_task_claim
wf_task_submit
```

Claim orderingは`07`。`claim_next`も同じ。

## 8. Human Review

```text
wf_review_list
wf_review_info
wf_review_submit
```

Submit first-wins。

## 9. Artifact / Log / Runner

```text
wf_artifact_info
wf_log_read
wf_runner_info
```

`wf_artifact_info`はmetadata/history。Managed Artifact実体downloadをMVP public Service必須にはしない。Workflow内利用はRuntime Handle `artifact_materialize`。

Log readはattempt_id + offset/tail。External path不可。

## 10. MCP namespace

```text
Core: wf_start
Novel: novel_wf_start
FX: fx_wf_start
```

`system_namespace`必須。親はtool subsetを非公開可能。

MVP public tool候補:

```text
<ns>_wf_start
<ns>_wf_list
<ns>_wf_info
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

## 11. Error model

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

## 12. Idempotency

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

## 13. Validation order

State change:

1. request schema
2. Actor/AccessScope
3. Authorization
4. idempotency lookup/reservation
5. domain validation
6. side effect + Event + result transaction
7. response

Secret値をrequest hash/Eventへ保存しない。

## 14. Read authorization / pagination

`wf_list/info/output_*/task_info/review_*/artifact_info/log_read/runner_info` 全てauthorize。

Listはfiltered scope適用。

List/Event/Artifact historyはopaque cursor pagination。

## 15. Python / Web Adapter

Python APIも同じService経路。

HTTP AdapterはService modelをREST/JSONへmappingする薄い層。Exact HTTP pathはWebUI/API設計時に確定。

## 16. Observability

State change Eventへactor/source/request_id。Request body/Secret/巨大payloadを無条件保存しない。

## 17. 受入条件

1. namespaced MCP
2. all read/write Authorization
3. `wf_info` Output body無し
4. output info/read inline/blob透過
5. output JMESPath select
6. MCP large response fail-closed/no truncate
7. large Log separate read
8. task/review operations
9. child direct control reject
10. idempotency scope/TTL
11. cursor pagination
