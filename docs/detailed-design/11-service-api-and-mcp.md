# 11. Service API / MCP 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/03-runtime-and-scheduling.md`
  - `docs/detailed-design/07-external-and-human.md`
  - `docs/detailed-design/10-retry-recovery-cancel.md`
  - `docs/detailed-design/12-security-and-secrets.md`

## 1. 目的

本書は JobRunner の Runtime Service API、Python API、MCP Adapter、Web Adapter が共有する操作境界、request/response、idempotency、namespace、error handlingを定義する。

## 2. 基本原則

1. Web / MCP / Python APIは同じService layerを呼ぶ。
2. Adapterがrepository / SQLiteを直接更新しない。
3. state-changing operationはAuthorizationProviderを必ず通す。
4. state-changing operationはoptional idempotency keyを受け付ける。
5. MCP public tool名は親システムnamespaceを必須とする。
6. read APIは大きいlog本文を通常infoに埋め込まない。
7. responseはJSON-compatibleなtyped modelとする。

## 3. Service構成

概念:

```text
WorkflowService
JobService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
RunnerService
```

単一Facadeにまとめてもよいが、内部責務は分離する。

## 4. ActorContext

すべてのService operationは内部的にActorContextを受け取れる。

```text
actor_type
actor_id optional
source optional
access_scope optional
metadata optional
```

MVP default providerがAllowAllでも呼び出し経路は省略しない。

## 5. 共通request metadata

state-changing operation:

```text
request_id optional
actor_context
```

`request_id`はidempotency keyとして扱う。

## 6. Workflow operations

### 6.1 start

logical name:

```text
wf_start
```

request:

```text
workflow reference
inputs
priority optional
source_identity optional
request_id optional
```

response:

```text
workflow_run_id
status
created_at
validation summary
```

開始前validation failure時はWorkflow Runを作らない。

### 6.2 list

```text
wf_list
```

filter候補:

```text
workflow_id
status
conclusion
created_from / created_to
limit
cursor
```

### 6.3 info

```text
wf_info
```

option:

```text
include_jobs
include_attempts
include_steps
include_artifacts
include_events_summary
```

Execution Log本文は含めない。

### 6.4 pause / resume / cancel

```text
wf_pause
wf_resume
wf_cancel
```

request_id対応。

terminal Runに対する無効操作はconflictまたはidempotent no-opをoperation規則で統一する。

### 6.5 retry

```text
wf_retry
```

MVPではJob指定を基本とする。

request:

```text
workflow_run_id
job_run_id
request_id optional
```

Retry Input変更は許可しない。

## 7. External Task operations

### task_info

```text
wf_task_info
```

read-only。

### task_claim

```text
wf_task_claim
```

request候補:

```text
claimed_by
workflow_run_id optional
job key/filter optional
request_id optional
```

response:

```text
task_id
lease_id
lease_expires_at
workflow_run_id
job_run_id
attempt_id
input
metadata
```

### task_submit

```text
wf_task_submit
```

request:

```text
task_id
lease_id
result
claim_next optional
request_id optional
```

## 8. Human Review operations

### review info

read APIとして`wf_info`内のwaiting_review一覧または専用read operationを提供可能。

### review submit

```text
wf_review_submit
```

request:

```text
workflow_run_id
job_run_id
attempt_id
outcome: approve | reject
comment optional
request_id optional
```

## 9. Artifact operations

```text
wf_artifact_info
```

request:

```text
artifact_id
```

またはworkflow/job/nameから検索可能。

responseはArtifact reference metadataのみ。

Artifact実体downloadをCore標準APIに必須としない。

## 10. Log operations

```text
wf_log_read
```

request候補:

```text
attempt_id
offset optional
limit optional
tail_lines optional
```

response:

```text
content
next_offset
truncated
size_bytes
updated_at
```

## 11. Runner operations

```text
wf_runner_info
```

取得内容:

```text
runner pools
runner status
runner_instance_id
last_heartbeat_at
current job
restart count
restart suppressed
```

Runner管理用force kill APIはMVP公開Serviceに含めない。

## 12. Priority update

Workflow Run priorityは実行中でも変更可能。

logical operation候補:

```text
wf_priority_update
```

実行中Jobをpreemptしない。

Job priority個別変更をMVP必須にはしないがService内部拡張可能にする。

## 13. MCP namespace

Core logical nameとMCP public nameを分ける。

例:

```text
Core: wf_start
Novel MCP: novel_wf_start
FX MCP: fx_wf_start
```

Adapter登録時に`system_namespace`を必須とする。

空namespaceは原則拒否。

## 14. MCP tool set

MVP候補:

```text
<ns>_wf_start
<ns>_wf_list
<ns>_wf_info
<ns>_wf_pause
<ns>_wf_resume
<ns>_wf_cancel
<ns>_wf_retry
<ns>_wf_task_info
<ns>_wf_task_claim
<ns>_wf_task_submit
<ns>_wf_review_submit
<ns>_wf_artifact_info
<ns>_wf_log_read
<ns>_wf_runner_info
```

親システムは不要toolを非公開にできる。

## 15. MCP tool granularity

巨大な万能tool 1個にまとめない。

read / state change / external task / human reviewを分ける。

ChatGPT / Agentが誤操作しにくいschemaを優先する。

## 16. include option

`wf_info`は必要に応じて下位情報を含める。

```text
include_jobs=false default
include_attempts=false default
include_steps=false default
include_artifacts=false default
```

下位includeが上位includeを必要とする場合、Adapterで自動補完してもよい。

## 17. Pagination

list / events / artifact history等はcursor paginationを基本とする。

offset paginationを内部実装しても外部contractはcursorへ抽象化可能にする。

## 18. Error model

共通error:

```text
code
message
retryable
field/path optional
details optional
```

HTTP status等はWeb Adapterがmappingする。

MCPではtool errorとして構造化dataを返す。

## 19. Error code分類

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
internal_error
```

domain codeをdetailsに追加可能。

## 20. Idempotency

対象:

```text
wf_start
wf_pause
wf_resume
wf_cancel
wf_retry
wf_task_claim optional
wf_task_submit
wf_review_submit
priority update
```

同じrequest_id + operation + scopeで同じrequestなら最初のresponseを再返却する。

同じkeyでrequest内容が異なる場合は`idempotency_conflict`。

## 21. Validation order

state-changing request:

1. request schema validation
2. actor/authentication-derived context取得
3. authorization
4. idempotency lookup
5. domain state validation
6. transaction
7. Event記録
8. idempotency result保存
9. response

Secret値をrequest hash / Event payloadへ平文保存しない。

## 22. Python API

親システムからService objectを直接呼べる。

例:

```python
run = workflow_service.start(
    workflow="analysis.yml",
    inputs={"document_id": 1},
    actor=actor,
)
```

MCP/Webと同じvalidation / authorization / idempotencyを通す。

## 23. Web Adapter

WebUI画面設計は別途。

HTTP AdapterはService modelをREST/JSONへmappingする薄い層とする。

MVPのexact pathはWebUI/API設計時に確定してよい。

Core詳細設計ではService operationをSource of Truthとする。

## 24. Versioning

Service model / MCP schemaには将来互換性を考慮する。

MVPでURLに`v1`を必須とするかはWeb Adapter時に決めるが、内部request/response modelは破壊変更を識別できるversioning方針を持つ。

## 25. Observability

state-changing operationはactor/source/request_idをEventへ残す。

request body全体を無条件にEventへ保存しない。Secrets /巨大payloadを避ける。

## 26. 受入条件

1. Python API start
2. MCP namespaced start
3. namespace衝突防止
4. wf_info include options
5. large log separate read
6. task claim/submit
7. review submit
8. pause/resume/cancel/retry
9. idempotency replay
10. idempotency conflict
11. authorization hook
12. error mapping
13. parentによるtool非公開
14. pagination
15. priority update non-preemptive
