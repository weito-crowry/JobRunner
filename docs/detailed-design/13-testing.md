# 13. Testing 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`〜`12`

## 1. 基本原則

1. 状態遷移はintegration test必須。
2. SQLite concurrency/transactionは実DB。
3. Runner/Action Runnerは実Process E2E。
4. External/Human/Retry/Recoveryはnegative case必須。
5. Definition/Expressionはtable-driven。
6. 時刻依存はClock abstraction。
7. 競合testはbarrier/hook。
8. 完了判定はcoverage率より各詳細設計の受入条件対応を優先。

## 2. Foundation dependencies / package

Python3.10で:

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

確認:

- CEL custom function bindingで`jmespath(...)`
- Windows/Linux Python3.10
- dependency license inventory
- ruamel.yaml duplicate key reject/merge key explicit reject
- base packageのみでMCP/HTTP optional依存不要
- `[mcp]`, `[web]`, `[all]` extras
- Core -> Adapter逆依存無し

## 3. Definition / Expression / Type boundaries

- YAML1.2
- canonical-json-v1 golden bytes/hash
- Draft2020-12 only
- duplicate/merge/custom tag/unknown key reject
- Input nullable/default/null/extra
- env literal-only
- needs/Dynamic parent cycle
- executor field conflicts
- Action/Validator ID/version non-empty
- unknown Action/Validator start failure
- Validator executor constraints
- concurrency group case-sensitive
- signed64 boundaries
- NaN/Infinity reject
- timeout executor constraints
- invalid CEL/JMESPath
- missing/null strictness
- skipped dependency default skip
- continue-on-error effective success

### Definition reload

- valid YAML更新をprocess restart無しで認識
- mtime_ns+size変化cache replace
- explicit `WorkflowResolver.refresh()`
- invalid new YAMLはold cacheへsilent fallbackしない
- running Runはold snapshot継続
- Action/Validator code hot reloadはCore対象外

## 4. Integration Bootstrap / Process boundary

- importable dotted bootstrap entrypoint
- `role=parent|runner|action_runner`
- parent roleでRegistry/Auth/Secrets/Storage factory構築
- runner roleでValidator/Auth/Secrets/Store再構築
- action_runner roleでAction callable解決
- Windows spawnでもparent memory pickle依存無し
- Action RunnerへDB path/AuthProvider/SecretsProvider/Store credential非注入
- current Attempt materialized SecretだけIPC注入
- Parent/Runner/Action Runner Action version mismatch fail-closed
- Parent/Runner Validator version mismatch fail-closed
- bootstrap exceptionでScheduling開始しない

## 5. Custom Validator

Internal:

- JSON Schema -> Validator -> success_if
- valid/invalid/custom failure/exception
- result変換無し
- persistent Inputのみ、Secret/Runtime Handle無し

External:

- submit Validator valid/invalid/exception
- invalid submit terminalizes current Attempt
- retryable Validator failure -> new Attempt/Task

Reusable/Dynamic:

- Child Validator versions snapshot
- missing version fail-closed
- Dynamic preflight before insert
- reuse key includes Validator identity

## 6. JSON Output / PayloadStore

- null/boolean/number/string/array/object
- Workflow Output object
- optional Schema
- threshold-1/threshold/threshold+1
- multi-MiB success
- downstream inline/blob same value
- Workflow Output spill
- crash orphan cleanup
- payload missing/digest mismatch
- Attempt history immutable

## 7. Service / MCP / HTTP Adapter

Canonical names/contract:

- Definition and Run API separated
- no `wf_list/wf_info`
- namespaced MCP collision reject
- Run info body無し
- output info/read/select
- response_too_large no truncate
- exact HTTP v1 routes/methods
- workflow_ref query supports slash
- Idempotency-Key mapping
- status 200/201/400/401/403/404/409/413/500, no422
- idempotency replay original status/body
- canonical request/response fields

Forbidden API:

- no `wf_task_heartbeat`
- no lease renew/extend/transfer
- no `wf_job_skip`
- no failed/cancelled -> success override
- no generic conclusion mutation
- no completed Review rewrite
- submit+claim_next idempotency replay does not claim another Task

## 8. SecretGuard

- SecretsProvider non-empty str only
- invalid type / missing
- Output/state/Artifact metadata/Event/error reject
- spill前guard
- log redact
- managed Artifact byte match/chunk boundary
- transformed Secret guarantee外

## 9. Runner Pool / Scheduling / Maintenance

Runner Pool:

- registered Pool only
- default pool resolution
- unknown Pool pre-start reject
- runner_count exact process count
- invalid runner_count reject
- heartbeat/lost/stale relation validation
- restart mode/window/backoff
- restart suppression
- no per-Pool Action allow-list
- no Pool global pause

Scheduling:

- Workflow/Job priority
- Dynamic order rank/source
- Pool-matched claim
- one internal running/Run
- multiple Runs parallel
- non-preemptive priority
- claim race exactly one
- External same ordering
- concurrency case sensitivity

Maintenance:

- no busy loop
- earlier deadline wake
- default max sleep5
- retry due without traffic
- Lease expiry without client traffic
- Pause中Lease expiry継続/retry activation停止
- Resume due retry
- restart overdue retry/Lease/Retention
- repeated due processing idempotent

## 10. Runner / IPC

- heartbeat/main-loop stall/lost exactly once
- heavy Action heartbeat
- Runtime Handle request correlation
- state/artifact request-response
- ActionFailure propagation
- request wait中cancel
- large result small IPC frame
- path/digest reject
- child crash/hang
- Parent shutdown vs Workflow cancel
- old Runner fencing/restart suppression

## 11. Retry / Failure policy

- retry absent=max1/disabled
- `retry:{}`=max2/retryable/zero backoff
- numeric validation/backoff formula
- core retryable map
- ActionFailure/Validator retryable preserve
- Automatic Retry failed Attempt only
- retry condition error preserves original failure
- manual retry Input Snapshot/version eligibility
- pre-Attempt failure new Run
- run_attempt++
- target Input exact copy
- descendants reactivation/reuse

## 12. Timeout

- internal unlimited default
- internal timeout cancel/terminate/retryable
- external/human/reusable timeout reject

## 13. Dynamic Jobs

Basic:

- 0/1/N
- stable/index key
- percent encoding
- no fixed job_key length
- 1000/1001 rollback
- 2/3+ depth
- parent cycle
- Action+Validator preflight

Ordering:

- omitted order -> source order
- literal `source_order`
- list `{expr,direction}`
- asc/desc/default asc
- invalid unknown key/type/NaN/Infinity
- lexicographic criteria + source tie
- order_rank snapshot/recovery

Outcome/group:

- Root `if=false` -> skipped
- Root `foreach=[]` -> success
- Nested parent success+0 -> success
- Nested parent skipped+0 -> skipped
- Nested parent failure/blocked+0 -> blocked
- Nested parent cancelled+0 -> cancelled
- `always()` does not synthesize missing parent item
- mixed skipped/success -> success
- failure/blocked/cancel precedence
- dynamic_expansions root/nested partial unique
- skip vs empty persisted/recovered distinctly

## 14. Reusable Workflow

- registered/relative ref
- traversal/symlink/non-filesystem rules
- Definition+Action+Validator binding
- Retry same binding
- Parent/Child state isolation
- parent -> child ArtifactRef via `with`
- child -> parent ArtifactRef via Workflow Output
- no automatic child Artifact mirror
- cross-run ArtifactRef fixed on Retry
- direct Child control reject
- cycle/Dynamic/restart duplicate

## 15. ArtifactStore / ArtifactRef

Managed:

- durable put/generation/materialize
- workdir path safety
- DB failure orphan cleanup

ArtifactRef:

- canonical shape
- DB re-resolve caller metadata
- same-run positive
- cross-run requires persistent Input ref + Authorization
- raw ID only reject
- forged ref reject
- retained/deleted data reject
- persistent Input ref changes digest
- Input外materialize reuse ineligible

External:

- no Core fetch/materialize/delete

## 16. External / Human

External:

- arbitrary result/spill
- one Task/Attempt/active Lease
- claim race/order/claim_next
- no heartbeat/renew/extend/transfer
- Lease requeue/fail/overdue/Pause/Recovery
- stale/cancel submit reject

Human:

- pending/approve/reject
- Output null
- first-wins
- Retry creates new Review
- no timeout/Validator
- failed->success override reject
- manual skip operation absent
- completed Review rewrite reject

## 17. Same-Run Result Reuse

- same Run only, no cross-Run auto cache
- Input/direct Artifact/Definition/executor/Action/Validator key
- explicit cross-run ArtifactRef in persistent Input allowed and fixed
- changed component/missing Payload/state.get/Input外Artifact materialize negative
- successful descendant match reuse
- mismatch -> new Run

## 18. Recovery / Persistence / Idempotency

- Parent restart runner_lost
- queued/waiting/backoff restore
- overdue processing
- Dynamic/Child duplicate無し
- reuse pending restore
- completed Run no Recovery reopen
- migration/WAL/FK/future schema reject
- Dynamic expansion outcomes/unique indexes
- Reusable/Task/Lease/Review uniqueness
- state current/history atomic
- idempotency Actor/Scope/TTL/expired replace
- HTTP original status replay

## 19. Retention

Policy:

- System all-null unlimited
- Workflow field override
- effective policy Run snapshot
- later config change no effect on existing Run

Cutoff/safety:

- run-history completed_at/non-terminal retained
- Log completed/close/running retained
- normal Event created_at
- Artifact data/metadata created_at + owner non-terminal guard
- Payload follows Run history
- Managed data before metadata
- External data never delete
- FK leaf->parent
- retained-away data not current/materializable
- reuse fail-closed

Audit:

- system-level retention Event has no Run FK
- excluded from event-days
- survives Run delete
- no duplicate audit
- orphan cleanup audit

## 20. Authorization / Security

- AllowAll/Deny/filtered scope
- all public read/write authorize
- Provider reconstructed by Integration Bootstrap
- Action Runner no provider credential
- log path safety
- Artifact URI no fetch
- cross-run Artifact explicit ref + authorization
- Reusable no scope escalation
- arbitrary shell無し

## 21. Platform / CI

Windows11/Linux。

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform-matrix
```

## 22. MVP completion gate

1. `01`〜`12`受入条件対応
2. dependencies/package extras
3. Integration Bootstrap Windows spawn
4. YAML reload/canonical JSON/JSON Schema
5. Validator
6. Runner Pool
7. migration/Dynamic uniqueness/FK/Retention
8. process/IPC
9. claim/concurrency/deadline races
10. PayloadStore
11. Service/MCP/HTTP + forbidden API boundary
12. ArtifactStore/ArtifactRef
13. External/Human
14. Dynamic1000/nested/order/skip-empty
15. Reusable binding/artifact mapping
16. SecretGuard/Authorization
17. Retry/Recovery
18. same-run Result Reuse
19. idempotency replay
20. Retention sweep/audit/orphan cleanup

WebUI E2Eは後続。
