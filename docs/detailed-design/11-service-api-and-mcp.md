# 11. Service API / MCP 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 基本原則

1. Web/MCP/Python APIは同じService layerを呼ぶ。
2. AdapterはDBを直接更新しない。
3. public read/write operationはAuthorizationProviderを通す。
4. state-changing operationはoptional `request_id`を受ける。
5. MCP public tool名は親システムnamespace必須。
6. 大きいExecution Log本文はinfoへ埋め込まない。

## 2. Service

```text
WorkflowService
JobService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
RunnerService
```

## 3. ActorContext / AccessScope

各Service operationはActorContextとAccessScopeを持つ。

Adapterは親認証結果からcurrent principalを構築する。

state-changing public callでactor_idが無い場合、Adapterはclient/session principalを補う。anonymous global principalでidempotency keyを共有しない。

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

request:

```text
workflow reference
inputs
priority optional
source_identity optional
request_id optional
```

開始前validation failureではRunを作らない。

### `wf_retry`

MVPは `workflow_run_id + job_run_id` 指定。

- failed Jobのみ
- Runがcompleted/failureならexplicit reopen
- success/cancelled Runはreject
- Input変更不可

Child Workflow Runへのpublic retry/pause/resume/cancel/priority updateは禁止。

## 5. External operations

```text
wf_task_info
wf_task_claim
wf_task_submit
```

`wf_task_claim` は `07` のcandidate orderingを使用。

`claim_next` も同じselection rule。

## 6. Human operations

```text
wf_review_list
wf_review_info
wf_review_submit
```

`review_submit` は first-wins。

## 7. Artifact / Log / Runner read

```text
wf_artifact_info
wf_log_read
wf_runner_info
```

Artifact実体downloadはCore必須機能にしない。

Log readはattempt_id + offset/tail等で行い、外部pathを受け取らない。

## 8. MCP namespace

Core logical nameとpublic nameを分離。

```text
Core: wf_start
Novel: novel_wf_start
FX: fx_wf_start
```

Adapter registration時 `system_namespace`必須。空はreject。

親システムは不要toolを非公開にできる。

## 9. `wf_info`

option:

```text
include_jobs
include_attempts
include_steps
include_artifacts
include_events_summary
```

Execution Log本文は含めない。

## 10. Error model

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
child_run_direct_control_forbidden
internal_error
```

## 11. Idempotency

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

### 11.1 request identity

Canonical identity:

```text
scope + operation + request_id
```

`scope` は `08` の正規規則で以下を含む。

- system namespace
- resource scope
- AccessScope canonical identity
- Actor/client principal isolation key

異なるActor/AccessScopeで同じrequest_idを共有しない。

### 11.2 replay

TTL内:

- same request hash -> first response replay
- different request hash -> `idempotency_conflict`

TTL後:

- expired rowが残っていても新requestとして再利用可能
- Persistenceが同transactionでexpired rowを置換して新side effect/resultを確定

### 11.3 validation order

1. request schema
2. Actor/AccessScope構築
3. Authorization
4. idempotency lookup/reservation
5. domain state validation
6. transaction side effect + Event + result
7. response

Secret値をrequest hash/Eventへ平文保存しない。

## 12. Read authorization

`wf_list/info/task_info/review_list/review_info/artifact_info/log_read/runner_info` もAuthorizationProviderを通す。

ListはProviderが返したfiltered scopeをqueryへ適用する。

## 13. Pagination

list/events/artifact history等はcursor paginationを外部contractとする。

Cursorはopaque。ordering key + id tie-breakを含む。

## 14. Python API / Web Adapter

Python APIも同じvalidation/authorization/idempotency経路。

HTTP AdapterはService modelをREST/JSONへmappingする薄い層。Exact HTTP pathはWebUI/API設計時に決める。

## 15. Observability

State change Eventには actor/source/request_id を残す。Request body全体やSecret/巨大payloadは無条件保存しない。

## 16. 受入条件

1. namespaced MCP
2. all public read/write authorization
3. wf_info include
4. large log separate read
5. task claim selection統一
6. review list/info/submit
7. child direct control拒否
8. idempotency actor/scope isolation
9. TTL内 replay/conflict
10. TTL後 expired row残存時key再利用
11. cursor pagination
