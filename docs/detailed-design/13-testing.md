# 13. Testing 詳細設計

- Status: Draft v2.4
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
9. Schema/IPC/Service contractはgolden + negative testで固定。

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

- CEL custom function `jmespath(...)`
- Windows/Linux Python3.10
- dependency license inventory
- YAML duplicate/merge reject
- base packageにMCP/HTTP optional依存不要
- `[mcp]`, `[web]`, `[all]`
- Core -> Adapter逆依存無し

## 3. Definition / System baseline / Priority / Registry

Definition:

- YAML1.2
- canonical-json-v1 golden
- Draft2020-12 only
- duplicate/merge/custom tag/unknown key reject
- Core/Pydantic strict no-coercion
- bool not accepted as integer/number
- string not coerced to number/bool
- Input nullable/default/null/extra
- env literal-only
- needs/Dynamic parent cycle
- executor omitted -> internal
- uses -> reusable / explicit executor forbidden
- Job `outputs.schema` exact shape
- Human outputs.schema reject
- Reusable outputs.schema allow
- Action/Validator ID/version non-empty
- same ID duplicate registration reject
- one current version semantics
- snapshot/current mismatch fail-closed
- concurrency case-sensitive
- concurrency scope=`workflow_id + group`
- mixed `max-runs` uses conservative minimum capacity
- signed64/NaN/Infinity boundaries
- timeout executor constraints

System baseline:

- Root start exact `system_workflow_defaults_json`
- default runner, dynamic/lease/output/log/progress + retention baseline
- idempotency TTLはRun baseline外
- Workflow setting > Run System baseline > canonical default
- Root restart stored baseline
- Reusable Child inherits Parent Run baseline, not current System
- Nested Child same lineage baseline
- Child effective settings/Retention binding snapshot

Priority:

- root `wf_start.priority` override
- omitted -> Definition priority
- Child priority=current root
- root update all non-terminal descendants
- future Child updated priority
- Child direct reject
- no preemption

Definition Resolver/reload:

- executable Workflow refはUTF-8 YAML source bytesをResolverから取得可能
- registered Workflow IDもtyped objectだけではなくsource bytes必須
- valid update without restart
- explicit refresh
- `wf_start` always re-reads current source bytes
- unchanged mtime/size but changed bytes still detected at start
- invalid/unavailable new source no old-cache fallback
- existing Run old snapshot

## 4. Expression / Context / Persistent Input / Secret binding

Context:

- `workflow={id,name,version}` exact
- `run={id}` exact
- `job={template_id,key,executor}` exact
- Dynamic template `job.key=null`
- unavailable field error
- state missing != null
- field allow/deny matrix
- success_if forbids needs/state/secrets/failure
- retry.if failure context

Persistent Input:

- `$base` shallow merge
- final object only
- missing/null strictness
- RFC6901 Secret binding pointer
- sort/duplicate reject
- full-scalar Secret only
- marker-like unbound literal
- input digest golden
- Secret value excluded
- Secret binding付きAttemptはreuse_eligible=false

Activation:

- internal pending before claim
- external/human/reusable initial Attempt direct
- Retry all executor pending
- exact copy
- Resume/restart no ready Input re-evaluation

## 5. Integration Bootstrap / Action / Validator invocation

Bootstrap:

- importable dotted entrypoint
- role parent|runner|action_runner
- Parent Registry/Auth/Secrets/Store
- Runner Validator/Auth/Secrets/Store
- Action Runner Action callable only
- Windows spawn/no parent callable pickle
- no DB/provider/storage credential in Action Runner
- current Attempt Secret only
- bootstrap version mismatch fail-closed
- bootstrap exception prevents scheduling

Action:

- `uses_runtime=false` -> exactly one positional execution Input
- `uses_runtime=true` -> Input + Runtime Handle
- invalid arity registration reject
- sync return JSON
- async function
- sync function returning awaitable
- varargs still follows explicit `uses_runtime`
- Action Runner owns await event loop

Validator:

- sync-only callable `(result,persistent_input)`
- canonical ValidationResult defaults
- invalid=false missing code -> domain_validation_failed
- returned awaitable -> validator_contract_error
- exception -> validator_exception

## 6. Authorization / Runtime Handle / SecretGuard

Authorization:

- all public read/write authorize
- state_get -> workflow_state.read
- state_set -> workflow_state.write
- Artifact put/register -> artifact.create
- Artifact materialize -> artifact.read
- denied no side effect
- telemetry Attempt fencing

State runtime:

- state_set current+history same DB transaction
- write visible immediately after response
- later Attempt failure/cancel/timeout/runner_lost does not rollback
- history records producer Job/Attempt/current open Step or NULL
- state_get marks Attempt reuse ineligible
- state_set marks Attempt reuse ineligible

SecretGuard:

- SecretsProvider non-empty str only
- unique Secret name resolved once per Attempt
- same resolved value used for all bindings/redaction/artifact scan
- Output/state/Artifact metadata/Event/error reject
- spill pre-guard
- stdout/stderr streaming redaction catches chunk-boundary match
- log/progress/step telemetry redaction
- managed Artifact byte match/chunk boundary
- transformed Secret guarantee外
- Input read references only

## 7. Result validation

Internal/External:

- JSON -> Schema -> Validator -> success_if -> SecretGuard -> PayloadStore
- valid/invalid/custom failure/exception
- Validator persistent Input only

Reusable:

- Child output -> optional Parent Schema -> SecretGuard -> Parent PayloadStore
- Schema fail -> Parent failure
- Retry same binding

Human:

- outputs.schema forbidden
- approve Output exactly null
- reject no current Output

## 8. JSON Output / PayloadStore

- all JSON types
- Workflow Output object
- threshold boundaries
- multi-MiB
- downstream transparent
- Workflow spill
- crash orphan
- payload integrity
- Attempt immutable

## 9. Runner Pool / Scheduling / Priority / Run status

Runner Pool:

- registered only
- default from Run snapshot
- runner_count exact
- heartbeat/lost/stale
- restart/suppression
- no Action allow-list/Pool pause

Workflow Run status:

- admitted initial/Child Run=`running`
- `queued` only concurrency waiter
- paused admitted holder keeps slot
- paused concurrency waiter has no slot and is wake候補外
- resume holder -> running
- resume waiter -> queued/concurrency

Scheduling:

- static non-dynamic rows only at start
- Dynamic template no Job/Attempt
- initial Job ready_at NULL
- internal pending claim
- all-executor Retry pending
- noninternal due Retry no with re-eval
- Internal order uses Job `ready_at`
- External claim order uses Task `available_at`
- Static same-condition ordering uses `job_key` before opaque IDs
- Dynamic order_rank/source_order stays ahead of job_key
- Lease requeue updates Task `available_at`
- Child priority propagation
- one internal/Run
- multiple Runs
- claim races
- concurrency scope/workflow separation

## 10. Maintenance / Recovery

- no busy loop
- earlier deadline wake
- max sleep5
- due boundary `now >= deadline`
- retry/Lease due without traffic
- Pause lease continues/retry waits
- restart overdue/Retention
- pending survives restart
- runner_lost
- root/Child priority repair
- Workflow status/wait_reason + concurrency holder/waiter repair
- completed no reopen except explicit Manual Retry
- orphan scanner does not delete unowned object younger than `orphan_cleanup_grace_seconds`
- default grace300 / invalid `<=0` reject

## 11. Runner / IPC / Step

- handshake/envelope/terminal matrix
- exact Attempt persistent_input
- Secret bindings/materialization
- unique Secret resolution once/Attempt
- marker literal
- pre-child Secret failure
- Runtime request correlation/cancel
- state/artifact auth request-response
- state_set immediate/nonrollback
- streaming stdout/stderr redaction across chunks
- progress/log/step telemetry redaction
- at most one open Step per Attempt
- second/nested `step_started` reject while Step open
- step_key correlation
- Step `start_metadata_json` / `finish_metadata_json` separate
- state history associates current open Step
- large result file
- Validator persistent Input only
- fencing
- cancel/result race first terminal wins
- timeout deadline result rejection/grace cleanup only

## 12. Execution Log / Event / Progress

Execution Log:

- all four executors
- Run-snapshotted normal/debug
- System/source change after Run start no effect
- no Input/Output auto dump
- deleted explicit unavailable

Event:

- transitions append-only
- `dynamic_index_key_fallback` exactly once per fallback expansion
- explicit dynamic key emits no fallback Event
- retention audit exclusion
- wf_event_list

Progress:

- Job mode snapshot none/explicit/auto
- terminal fraction1
- Reusable mirrors Child
- Workflow mode Run snapshot
- Workflow concrete Job average
- Dynamic template denominator外
- generated growth may lower percent
- no conclusion effect

## 13. Retry / Result Reuse

Retry:

- absent/max1
- empty block/max2
- failed Attempt only
- no new Attempt at schedule
- exact pending copy
- Retry queued target clears `conclusion/completed_at`
- pre-Attempt new Run
- completed failure Run explicit reopen increments run_attempt
- non-terminal retry does not increment run_attempt
- target no with/item/version re-eval
- Secret binding fixed/value rotates only on re-execution
- completed Run Manual Retry reacquires concurrency slot
- concurrency reject leaves Run/Job unchanged
- concurrency queue reopens `status=queued, wait_reason=concurrency`
- reopen clears Run conclusion/completed/failure/current Workflow Output

Strict reuse:

- blocked/skipped reactivation
- successful current if/input/artifact/version/Payload validation
- executor-specific output revalidation
- state_get successful Attempt ineligible
- state_set successful Attempt ineligible
- Secret-bound successful Attempt ineligible
- non-input Artifact materialize ineligible
- mismatch -> new Run

## 14. Dynamic Jobs

- no template Job/Attempt
- Root/Nested/3+ depth
- explicit stable key
- omitted key -> 0-based index exact identity
- one fallback warning Event/expansion
- no warning with explicit key
- no fixed job_key max
- generated limit from Run snapshot
- System limit change after Run start no effect
- 1000/1001 rollback
- ordering/stable tie
- group status queued/running/completed
- if=false vs empty
- zero-parent
- expansion digest
- changed source/key/item/order -> new Run
- whole skipped re-evaluate
- root/nested unique/recovery

## 15. Reusable Workflow

- every executable ref provides current UTF-8 YAML source bytes
- relative/registered reference/path safety
- first binding re-reads current Child source bytes
- invalid/unavailable current Child source fails binding; no stale cache fallback
- Binding Definition+versions+System baseline
- Child settings/Retention
- current System change no effect
- nested baseline
- Child priority=root
- root update
- source_identity/Actor/Scope
- state isolation
- ArtifactRef/no mirror
- Parent outputs.schema
- Parent progress
- Child concurrency scope uses Child workflow_id
- admitted Child=running
- Child concurrency queue creates Child queued/Parent waits
- Child concurrency reject creates no Child row and Parent Attempt fails
- Retry after reject uses same binding and does not reread source
- direct control reject
- cycle/depth/Dynamic/restart
- parent retention upper bound

## 16. ArtifactStore / ArtifactRef

- Managed Artifact immutable
- External Reference Artifact logically immutable by parent contract
- changing external content requires new Artifact identity/generation
- managed put/generation/materialize/path
- SHA-256 public digest format
- external digest format validation/no content verification
- canonical ref/DB resolve
- same/cross run auth
- raw/forged reject
- no source pin
- retained/deleted reject
- Input digest/ineligible runtime materialize
- External no Core data operation

Retention:

- managed data can expire before metadata
- metadata expiry forces managed data delete first
- metadata row deleted at metadata due
- wf_artifact_info not_found after row deletion
- external metadata deletion never deletes external data

## 17. External / Human

External:

- Job override > Run effective lease
- Task snapshot
- arbitrary result
- claim race
- claim order uses `available_at` then stable `job_key` before opaque IDs
- lease expiry boundary `now >= expires_at`
- no renew
- requeue/fail
- submit Retry later
- submit+claim_next one tx/replay
- task_info hides another claimant's lease_id
- self claimant may recover current active lease_id
- submit validates lease + claimant ownership

Human:

- approve/reject/null Output
- reject no output
- first-wins
- Retry new Review
- no timeout/Validator/Schema/override/skip/rewrite

## 18. Persistence schema

Verify all18 tables:

- exact columns/checks/indexes/FKs/Repository invariants
- canonical fixed UTC timestamp format
- Workflow queued/running/paused/completed + wait_reason/slot invariant
- System baseline/effective settings
- Child binding baseline/settings/Retention
- priority support
- Dynamic no Job row
- pending invariant + retry target terminal fields clear
- Secret bindings/digest + secret-bound reuse_eligible=0
- one-running internal/Run
- one-running Step/Attempt partial unique
- Dynamic unique/digest
- State current/history atomic + current Step producer
- Artifact/log schema
- Runner liveness
- External Task config/Lease
- Human immutable
- Idempotency
- Reusable reject no Child row
- Child-first Retention
- migration gap/future

## 19. Service / MCP / HTTP

- Core Service models strict/no-coercion
- HTTP query explicit parse only
- canonical timestamp response/filter normalization
- admitted start=running / queue=queued
- pause/resume holder vs waiter
- Run info jobs/Dynamic groups/attempts/Step exact shape
- active_task/review/child IDs navigation
- Input info/read refs only
- Output/Event/Log
- State list/read/history read-only public API
- no public State mutation API
- priority start/update + PATCH exact body
- namespaced MCP
- exact HTTP incl Input/Output/State/Event
- Log byte offset boundary validation
- status/no422/idempotency replay
- concurrency_limit_reached 409
- forbidden APIs

## 20. Idempotency / Concurrency

- request hash
- Actor/Scope isolation
- fast read advisory
- BEGIN IMMEDIATE recheck
- replay/conflict/expiry
- no reserved
- claim exactly one
- submit claim_next replay
- concurrency scope=`workflow_id + group`
- holder=`running` or admitted paused
- queued/paused waiter no slot
- mixed max-runs conservative capacity
- waiter head-of-line fairness
- Manual Retry concurrency reacquire atomicity

## 21. Retention

- Run System baseline retention snapshot
- Workflow override
- Child inherited baseline + Child override
- later config no effect
- completed/nonterminal cutoffs
- all executor Logs
- Events/audit
- Artifact data/metadata precedence
- cross-run no pin
- Payload run history
- Child depth-first
- audit survives
- orphan cleanup grace/audit
- retention loss fail-closed

## 22. Platform / CI

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

## 23. MVP completion gate

1. `01`〜`12`全受入条件対応
2. Dependencies/extras/strict model parsing/source resolver
3. Definition/System baseline/Priority/Registry/reload
4. Expression context/Input/Secret binding/reuse safety
5. Bootstrap/Action invocation/IPC/Runtime Auth
6. Runner Pool/Scheduling/Run status/Concurrency
7. Exact18-table schema/migrations
8. Payload/Artifact immutability + retention
9. Validator/SecretGuard/streaming redaction
10. State immediate/reuse + public read APIs
11. External/Human/Lease ownership
12. Dynamic index/1000/nested/order/status/reuse
13. Reusable source/baseline/priority/concurrency/artifact/schema/progress
14. Retry/Recovery strict reuse/reopen/output invalidation
15. Service/MCP/HTTP/Input/Output/State/Event
16. Idempotency/concurrency/claim_next
17. Common Log/Progress/single-open Step metadata
18. Retention/orphan grace/audit

WebUI E2Eは後続。
