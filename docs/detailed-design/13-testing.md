# 13. Testing 詳細設計

- Status: Draft v1.0
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

## 2. Foundation dependencies

Python3.10で:

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

をinstall/import/代表実行。

確認:

- CEL custom function bindingで`jmespath(...)` helper
- Windows/Linux Python3.10
- dependency license inventory
- ruamel.yaml duplicate key reject/merge key explicit reject

## 3. Definition / Expression / Type boundaries

- duplicate/merge/custom tag/unknown key reject
- Input nullable false/true
- required+nullable
- default:null nullable true/false
- extra Input reject
- env literal-only
- bad needs / Dynamic parent cycle
- executor field conflicts
- Action/Validator ID/version non-empty string
- unknown Action/Validator start failure
- Validator internal/external allowed, human/reusable reject
- concurrency group string/null/empty/case-sensitive
- priority/version/count signed64 boundaries
- NaN/Infinity numeric reject
- external/human/reusable timeout reject
- invalid CEL/JMESPath
- null/missing strictness
- skipped dependency -> downstream default skip
- continue-on-error -> effective success
- Dynamic group / Nested parent helper
- order_by NaN/Infinity/mixed type reject

## 4. Custom Validator

Internal:

- JSON Schema -> Validator -> success_if order
- valid true
- valid false + custom code/message/details/retryable
- validator exception -> `validator_exception`, retryable=false
- result unchanged after Validator
- Validator receives persistent Input, not materialized Secret/Runtime Handle

External:

- task_submit Validator valid/invalid/exception
- invalid result terminalizes current Task/Lease/Attempt
- retryable Validator failure follows Retry policy with new Attempt/Task

Reusable/Dynamic:

- Child binding snapshots Validator versions
- missing Child Validator version on Retry fail-closed
- Dynamic expansion validates Validator version before any generated Job insert
- Same-Run Reuse key includes Validator identity

## 5. JSON Output / PayloadStore

Positive result shapes:

```text
null/boolean/number/string/array/object
```

Workflow Outputはobject。

- optional JSON Schema各shape
- threshold-1 inline
- threshold inline
- threshold+1 blob
- multi-MiB success
- downstream inline/blob same value
- Workflow Output spill
- blob temp -> rename -> DB
- DB commit failure orphan cleanup
- payload missing/digest mismatch
- Attempt history immutable

## 6. Service / MCP / HTTP Adapter

Canonical names:

- `wf_definition_list/info`
- `wf_run_list/info`
- no `wf_list/wf_info`
- namespaced MCP collision reject

Output:

- Run info body無し
- output info/read inline/blob
- current Job / Attempt / Workflow source resolution
- JMESPath select
- MCP response_too_large no truncate

HTTP:

- exact v1 routes/methods from`11`
- Definition info uses query `workflow_ref`, slash-containing IDs supported
- opaque generated IDs in path
- Idempotency-Key header mapping
- no duplicate request_id body
- status 200/201/400/401/403/404/409/413/500
- no 422
- idempotency replay preserves original 201/200 status/body
- source_identity non-empty string when present

## 7. SecretGuard

SecretsProvider:

- non-empty str success
- empty/bytes/number/object -> `secret_value_invalid`
- missing -> `secret_not_found`

Known Secret:

- Output substring/nested reject
- spill前reject
- state.set reject
- Artifact metadata/URI reject
- Event/error reject
- log redact
- managed Artifact byte match/chunk boundary reject
- transformed Secretは保証外

## 8. Scheduling / Claim / Maintenance

- Workflow/Job priority
- Dynamic order/source
- Pool routing
- one internal running / Run
- multiple Runs parallel
- pause/resume
- non-preemptive priority
- internal claim race exactly one
- External claim same ordering
- Concurrency group `Foo` != `foo`

Maintenance Loop:

- nearest-deadline waiting, no busy loop
- new earlier deadline wakes wait
- max sleep default5 seconds
- retry_not_before due without external event
- external Lease expiry without client traffic
- Pause中Lease expiry continues
- Pause中retry due does not start new Attempt
- Resume starts due retry
- restart processes overdue retry/Lease before normal scheduling
- repeated due processing idempotent

## 9. Runner / IPC

- heartbeat/main-loop stall
- Supervisor heartbeat scan lost exactly once
- heavy Action heartbeat
- start/ready/log/progress/step/error/exiting
- Runtime Handle correlation
- state/artifact request-response
- ActionFailure code/retryable/details
- unhandled exception retryable=false
- request wait中cancel
- result file protocol/large result frame small
- path traversal/size/digest reject
- child crash/hang
- Parent shutdown vs Workflow cancel
- old Runner fencing/restart suppression

## 10. Retry / Failure policy

Definition:

- retry block absent -> max1 / disabled
- `retry:{}` -> max2 / `failure.retryable` / zero backoff
- max-attempts >=2
- initial>=0/max>=initial/multiplier>=1 finite
- backoff formula attempt2/3/4 and cap
- retry condition type/error

Core retryable map:

- runner_lost true
- job_timeout true
- external_lease_expired true
- payload_storage_failed true
- action_exception/process_exit false
- protocol/result/schema/success_if false
- ActionFailure provided value preserved
- Validator provided value preserved
- version mismatch/human reject/secret/reuse/input unavailable false

Automatic:

- failed Attempt only
- max-attempts
- retry deadline created
- retry condition false
- condition error preserves original failure
- manual Attempt beyond max does not restart automatic budget

Manual:

- prior failed Attempt + Input Snapshot required
- pre-Attempt failure -> retry_input_unavailable/new Run
- Action/Validator version availability
- completed/failure Run reopen/run_attempt++
- success/cancelled Run reject
- target Input exact copy
- blocked/skipped descendants re-evaluate
- failed non-target no auto retry

## 11. Timeout

- internal no timeout long Action
- internal timeout cancel/terminate -> retryable true
- external/human/reusable timeout reject

## 12. Dynamic Jobs

- 0/1/N
- stable/index key
- parent same raw key
- percent encoding
- fixed job_key length limit無し
- 1000/1001 rollback
- 2/3+ arbitrary representative depth
- parent cycle
- parent+needs helper
- order
- Action+Validator preflight atomic rollback
- expansion crash/restart
- group status/conclusion
- arbitrary JSON aggregation
- full-key Artifact

## 13. Reusable Workflow

- registered ID
- caller-directory relative path
- traversal/symlink escape
- non-filesystem relative reject
- binding Definition+Action+Validator versions
- Retry same binding
- missing Validator version fail-closed
- Parent/Child Output/state isolation
- cycle
- direct Child control reject
- Dynamic+Reusable
- restart duplicate
- Child Output spill

## 14. ArtifactStore

Managed:

- durable put
- workdir traversal reject
- immutable generations
- materialize destination safety
- Retry current generation
- retention data delete
- store finalize DB failure orphan cleanup

External:

- URI metadata
- no fetch
- retention external data non-delete
- External LLM reference only

## 15. External / Human

External:

- arbitrary result/large spill
- one Task/Attempt
- claim race/order/claim_next
- Lease requeue/fail
- overdue Maintenance processing
- stale/cancel submit reject
- Pause/Recovery

Human:

- pending/approve/reject
- Output null
- first-wins
- cancel/pause/retry
- no timeout/validator

## 16. Same-Run Result Reuse

Scope same Run only, no cross-Run.

Key includes:

- persistent Input
- direct upstream Artifact identity
- Definition hash
- executor/Action version
- Validator identity/version

Negative:

- changed component
- Payload missing/digest
- state.get
- undeclared Artifact materialize

Manual Retry descendant:

- match -> reused event/success
- mismatch/ineligible/missing/version -> successful_job_result_not_reusable
- no automatic changed-Input re-execution
- new Run required

## 17. Recovery

- Parent restart running -> runner_lost
- queued/waiting/backoff restore
- overdue deadline processing
- Dynamic/Child duplicate無し
- reuse pending restore
- completed Run Recovery-only reopen無し

## 18. Persistence / Idempotency / Retention

- migration/WAL/FK/busy timeout
- future schema reject
- output inline/blob constraints
- Action/Validator snapshots
- retry/reuse metadata
- internal running unique
- Dynamic/Reusable/External/Human unique
- state current/history atomic
- deadline indexes
- concurrency race/case
- FK NO ACTION/explicit retention order
- idempotency Actor/AccessScope isolation
- TTL replay/conflict/expired replacement
- HTTP adapter_meta original status replay
- task_claim replay no extra Lease

## 19. Authorization / Security

- AllowAll/Deny/filtered scope
- all public read/write authorize
- Definition list/info authorize
- log path safety
- Artifact URI no fetch
- arbitrary shell無し

## 20. Platform / CI

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

## 21. MVP completion gate

1. `01`〜`12`受入条件対応
2. dependencies Python3.10 Windows/Linux
3. Validator contract
4. migrations/FK retention
5. process integration
6. claim/concurrency/deadline races
7. PayloadStore
8. Service/MCP/HTTP contract
9. ArtifactStore
10. External/Human E2E
11. Dynamic1000/nested/rollback
12. Reusable binding/cycle/version
13. SecretGuard
14. Retry failure policy
15. same-run Result Reuse
16. idempotency scope/status replay

WebUI E2Eは後続。
