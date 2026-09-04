# 13. Testing 詳細設計

- Status: Draft v2.1
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
- signed64/NaN/Infinity boundaries
- timeout executor constraints

System baseline:

- Root start snapshots exact `system_workflow_defaults_json`
- Includes default runner, dynamic/lease/output/log/progress + retention baseline
- idempotency TTL is not part of Run baseline
- Workflow setting > Run System baseline > canonical default
- Root restart uses stored baseline
- Reusable Child inherits Parent Run baseline, not current System config
- Nested Child continues same lineage baseline
- Child effective settings/Retention exact binding snapshot

Priority:

- root `wf_start.priority` overrides Definition priority
- omitted uses Definition priority
- Child Run priority=current root priority
- root priority update propagates all non-terminal descendants
- future Child inherits updated root priority
- Child direct priority update reject
- running Job not preempted

Definition reload:

- valid update without process restart
- explicit refresh
- invalid new YAML no old-cache fallback
- existing Run old snapshot

## 4. Expression / Context / Persistent Input / Secret binding

Context:

- `workflow={id,name,version}` exact
- `run={id}` exact
- `job={template_id,key,executor}` exact
- Dynamic template expression has `job.key=null`
- unavailable field is error, not implicit null
- state missing != null
- field-by-field allowed context matrix
- success_if forbids needs/state/secrets/failure
- retry.if failure context

Persistent Input:

- `$base` shallow merge
- final object only
- missing/null strictness
- RFC6901 Secret binding pointer
- pointer sorting/duplicate reject
- full-scalar Secret only
- marker-like unbound literal remains literal
- input digest golden=`{input,secret_bindings}`
- Secret value excluded

Activation:

- internal initial pending before claim
- external/human/reusable initial Attempt direct
- Retry all executor pending snapshot
- Runner/scheduler exact copy
- Resume/restart never re-evaluates ready pending snapshot

## 5. Integration Bootstrap / Process boundary

- importable dotted entrypoint
- role parent|runner|action_runner
- Parent Registry/Auth/Secrets/Store
- Runner Validator/Auth/Secrets/Store
- Action Runner Action callable only
- Windows spawn, no parent callable pickle dependency
- no DB/provider/storage credential in Action Runner
- current Attempt materialized Secret only
- bootstrap version mismatch fail-closed
- bootstrap exception prevents scheduling

## 6. Authorization / Runtime Handle / SecretGuard

Authorization:

- all public read/write authorize
- Runtime Handle `state_get` -> workflow_state.read
- `state_set` -> workflow_state.write
- Artifact put/register -> artifact.create
- Artifact materialize -> artifact.read
- denied Runtime operation no side effect
- telemetry uses Attempt ownership fencing

SecretGuard:

- SecretsProvider non-empty str only
- Output/state/Artifact metadata/Event/error reject
- spill pre-guard
- log redact
- managed Artifact byte match/chunk boundary
- transformed Secret guarantee外
- Input read returns references only, never materialized values

## 7. Custom Validator / Result validation

Internal/External:

- JSON -> Schema -> Validator -> success_if -> SecretGuard -> PayloadStore
- valid/invalid/custom failure/exception
- Validator gets persistent Input only

Reusable:

- Child success output -> optional Parent Job outputs.schema -> SecretGuard -> Parent PayloadStore
- Schema fail -> Parent Attempt failure/output_validation_failed
- Retry same binding

Human:

- outputs.schema forbidden
- approve Output exactly null
- reject has no current Output

## 8. JSON Output / PayloadStore

- null/boolean/number/string/array/object
- Workflow Output object
- threshold-1/threshold/threshold+1
- multi-MiB success
- downstream storage-transparent
- Workflow Output spill
- crash orphan cleanup
- payload missing/digest mismatch
- Attempt immutability

## 9. Runner Pool / Scheduling / Priority

Runner Pool:

- registered only
- default pool from Run snapshot
- runner_count exact auto-start
- invalid count
- heartbeat/lost/stale relation
- restart policy/suppression
- no per-Pool Action allow-list
- no Pool global pause

Scheduling:

- Run start static non-dynamic Job only
- Dynamic template no Job Run/Attempt
- initial queued ready_at NULL not claimable
- internal ready+pending claim
- all-executor Retry queued pending
- noninternal due Retry no with re-eval
- Workflow Run priority -> Job priority -> Dynamic order -> source -> ready -> ID
- Child priority propagation affects future selection
- one internal running/Run
- multiple Runs parallel
- internal/external claim races
- concurrency case sensitivity

## 10. Maintenance / Recovery

- no busy loop
- earlier deadline wakes
- default max sleep5
- retry due without traffic
- Lease expiry without traffic
- Pause Lease expiry continues / Retry activation stops
- Resume due Retry
- restart overdue Retry/Lease/Retention
- repeated due idempotent
- ready pending survives restart
- running old Runtime -> runner_lost
- root/Child priority repair
- completed Run no Recovery reopen

## 11. Runner / IPC

- ready -> start -> terminal -> exit
- malformed protocol/type
- duplicate ready/start/terminal
- terminal+exit matrix
- persistent_input exact Attempt snapshot
- Secret bindings/materialization exact
- unbound marker literal
- Secret missing/invalid pre-child
- Runtime request correlation
- cancel while waiting
- state/artifact request-response + auth deny
- log/progress/step validation
- large result file path/size/digest
- Validator persistent Input only
- stdout separate/redacted
- old Runner/late message fencing

## 12. Execution Log / Event / Progress

Execution Log:

- all four executors one Attempt Log
- normal/debug
- no automatic full Input/Output dump
- deleted file explicit unavailable

Event:

- state transitions append-only
- retention audit exclusion
- `wf_event_list` filters/order/pagination/resource chain validation

Progress:

- Job none/explicit/auto
- terminal fraction1 independent conclusion
- Reusable parent mirrors Child fraction
- Workflow averages concrete Job Runs only
- Dynamic template not denominator
- generated Job can lower percentage
- progress never affects conclusion

## 13. Retry / Result Reuse

Retry:

- absent=max1/disabled
- `retry:{}`=max2/retryable/zero backoff
- failed Attempt only
- no new Attempt at request/schedule
- exact pending snapshot copy
- due executor-specific Attempt
- pre-Attempt failure new Run
- run_attempt++
- target with/item/version not re-evaluated
- Secret binding fixed/value may rotate

Descendant strict reuse:

- blocked/skipped normal reactivation
- successful current if re-evaluation
- expected Input re-resolution
- Artifact/version/Payload check
- executor-specific stored Output revalidation
- mismatch -> new Run

## 14. Dynamic Jobs

- Template no Job Run/Attempt
- Root/Nested/3+ depth
- stable/index key
- no fixed job_key max
- System/Workflow generated limit
- 1000/1001 rollback
- ordering schema/stable tie
- if=false skipped vs foreach=[] success
- zero-parent propagation
- expansion_digest golden
- changed source/key/item/order -> not reusable
- expanded empty -> nonempty reject same Run
- whole skipped zero-job group may re-evaluate
- root/nested partial unique/recovery

## 15. Reusable Workflow

- registered/relative/path safety
- Binding Definition+Action+Validator versions
- Binding inherited System baseline
- Child effective settings/Retention
- current System change after Root start does not alter later Child
- Nested Child same baseline
- Child priority=root priority
- root update propagation
- source_identity/Actor/Scope inheritance
- Parent/Child state isolation
- ArtifactRef both directions/no mirror
- Parent outputs.schema
- Parent auto progress
- Child direct control reject
- cycle/depth/Dynamic/restart duplicate
- Parent run-history upper-bounds Child subtree

## 16. ArtifactStore / ArtifactRef

- managed put/generation/materialize/path safety
- orphan cleanup
- canonical ArtifactRef/DB re-resolve
- same-run
- cross-run requires persistent Input ref + Authorization
- raw/forged reject
- retained/deleted data reject
- no source retention pin
- Input ref changes digest
- non-input materialize reuse ineligible
- External no Core fetch/materialize/delete

## 17. External / Human

External:

- Job override > Run effective lease setting
- Task config snapshot/requeue fixed
- arbitrary result/spill
- one Task/Attempt/active Lease
- claim race/order
- no heartbeat/renew/transfer
- lease requeue/fail/Pause/Recovery
- submit validation schedules Retry later
- submit + claim_next + idempotency one transaction
- replay no double next claim

Human:

- approve/reject only
- Output null on approve
- reject Output unavailable
- first-wins
- Retry new Review via scheduler
- no timeout/Validator/Output Schema
- no success override/manual skip/rewrite

## 18. Persistence schema

Verify **all 18 tables** via SQLite PRAGMA/sqlite_master:

- exact columns/checks/indexes/FKs
- `workflow_runs.system_workflow_defaults_json`
- effective settings exact keys incl default_runner_pool
- Child binding system baseline/settings/Retention columns
- root/Child priority index/update support
- Dynamic template no Job row
- ready/pending all-executor invariant
- Attempt Secret bindings/input digest
- one-running internal
- Dynamic partial unique/digest
- state atomic
- Artifact/log schema
- Runner active slot/liveness
- External Task lease finite/config snapshot/active Lease unique
- Human Review immutable
- Idempotency no reserved/request hash/scope/TTL
- child-first retention
- migration gap/future reject

## 19. Service / MCP / HTTP

Run info:

- concrete jobs + Dynamic groups exact shapes
- attempts/steps/artifacts implication
- events_summary exact shape
- no large body embedding

Input:

- `wf_input_info/read` all selectors
- pending Job resolution wins over current Attempt
- unactivated Job unavailable
- JMESPath select
- Secret refs only
- MCP response-too-large no truncate

Output/Event/Log:

- selectors/read
- `wf_event_list`
- deleted Log error

Priority:

- start default/override
- root update descendant count/propagation
- Child direct reject

MCP/HTTP:

- namespaced collision
- exact v1 routes incl Input routes/events
- workflow_ref slash
- Idempotency-Key
- status 200/201/400/401/403/404/409/413/500
- no422
- original status/body replay

Forbidden:

- no lease renew
- no manual Job skip/success override
- no completed Review rewrite

## 20. Idempotency / Concurrency races

Idempotency:

- request hash excludes request_id/transport only
- Actor/AccessScope isolation
- fast read advisory
- BEGIN IMMEDIATE commit recheck
- same hash replay/different conflict
- expired replacement
- no reserved row
- task_claim exactly one
- submit claim_next exact replay

Concurrency:

- BINARY group race
- queue/reject
- holder release order

## 21. Retention

- System baseline retention snapshot
- Workflow overrides
- Child inherited System baseline + Child override
- later System change no effect
- completed/non-terminal cutoffs
- all executor Logs
- Event created_at/audit exclusion
- Artifact owner guard/no cross-run pin
- Payload follows Run history
- Child subtree depth-first delete
- audit survives deletion
- orphan cleanup audit
- retention loss makes materialize/reuse fail-closed

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

1. `01`〜`12`全受入条件対応表
2. Dependencies/extras
3. Definition/System baseline/Priority/Registry/reload
4. Expression context + persistent Input/Secret binding
5. Bootstrap/IPC/Runtime Handle Authorization
6. Runner Pool/Scheduling
7. Exact 18-table schema/migrations
8. PayloadStore/ArtifactStore
9. Validator/SecretGuard
10. External/Human E2E
11. Dynamic virtual group/1000/nested/order/reuse
12. Reusable baseline/settings/priority/artifact/schema/progress
13. Retry/Recovery strict reuse
14. Service/MCP/HTTP/Input/Output/Event contracts
15. Idempotency/concurrency/claim_next race tests
16. Common Log/Progress
17. Retention/audit/orphan cleanup

WebUI E2Eは後続。
