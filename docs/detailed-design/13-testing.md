# 13. Testing 詳細設計

- Status: Draft v2.0
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
9. Schema/IPC/Service contractはgolden + negative testで固定する。

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

## 3. Definition / Settings / Registry

- YAML1.2
- canonical-json-v1 golden bytes/hash
- Draft2020-12 only
- duplicate/merge/custom tag/unknown key reject
- Input nullable/default/null/extra
- env literal-only
- needs/Dynamic parent cycle
- executor omitted -> internal
- uses present -> reusable / explicit executor forbidden
- Job `outputs.schema` exact shape
- Action/Validator ID/version non-empty
- same ID duplicate registration reject
- one current version semantics
- snapshot/current version mismatch fail-closed
- concurrency case-sensitive
- signed64/NaN/Infinity boundaries
- timeout executor constraints
- invalid CEL/JMESPath
- skipped default dependency behavior
- continue-on-error effective success

Settings:

- System `default_runner_pool=default`
- custom System default pool
- omitted runs-on resolves Run snapshot default
- explicit runs-on overrides default
- System default changes after Run start do not change existing Run/generated Jobs
- Workflow > System hierarchy for Dynamic/lease/output/log/progress
- External Job > Workflow > System lease hierarchy
- idempotency TTL System-only
- effective settings JSON exact key set

Definition reload:

- valid file update without process restart
- explicit refresh
- invalid new YAML no old-cache silent fallback
- existing Run old snapshot

## 4. Expression / Persistent Input / Secret binding

Context matrix:

- each field allows exactly `02` contexts
- forbidden context compile/validation reject
- success_if cannot read needs/state/secrets
- retry.if failure context

Persistent Input:

- `$base` shallow merge
- final object only
- missing/null strictness
- RFC6901 Secret binding pointer
- pointer sorting/duplicate reject
- full-scalar Secret only
- marker-like unbound literal remains literal
- input digest golden=`{input,secret_bindings}` canonical hash
- Secret value excluded from digest

Activation:

- internal initial activation persists pending snapshot before claim
- external/human/reusable initial activation writes Attempt snapshot directly
- Retry all executor types writes pending snapshot
- Runner/scheduler exact copy to new Attempt
- Resume/restart never re-evaluates already ready pending snapshot

## 5. Integration Bootstrap / Process boundary

- importable dotted entrypoint
- role parent|runner|action_runner
- Parent Registry/Auth/Secrets/Store
- Runner Validator/Auth/Secrets/Store
- Action Runner Action callable only
- Windows spawn, no parent callable pickle dependence
- no DB/provider/storage credential in Action Runner
- current Attempt materialized Secret only
- bootstrap version mismatch fail-closed
- bootstrap exception prevents scheduling

## 6. Custom Validator / Result validation

Internal:

- JSON -> Schema -> Validator -> success_if -> SecretGuard -> PayloadStore
- valid/invalid/custom failure/exception
- Validator gets persistent Input with Secret refs, never Secret values
- result unchanged

External:

- submit Validator valid/invalid/exception
- invalid terminalizes current Attempt
- retryable failure schedules Retry but does not create inline new Task/Attempt

Reusable/Dynamic:

- binding Validator versions
- Dynamic preflight Validator version
- reuse identity includes Validator

## 7. JSON Output / PayloadStore

- null/boolean/number/string/array/object
- Workflow Output object
- optional Schema each shape
- threshold-1/threshold/threshold+1
- multi-MiB success
- downstream storage-transparent value
- Workflow Output spill
- blob temp/final/DB crash cases
- orphan cleanup
- payload missing/digest mismatch
- Attempt Output immutability

## 8. Runner Pool / Scheduling

Runner Pool:

- registered only
- runner_count exact auto-start
- invalid count
- heartbeat/lost/stale relation
- restart never/on_failure/window/backoff/suppression
- no per-Pool Action allow-list
- no Pool global pause

Scheduling:

- Run start creates static non-dynamic `job_runs` only
- Dynamic template has no Job Run/Attempt/Retry target
- initial static Job queued + ready_at NULL not claimable
- internal ready_at+pending required for claim
- all-executor retry queued pending invariant
- noninternal due Retry creates Attempt without with re-eval
- Workflow/Job priority + Dynamic order/source
- one internal running/Run
- multiple Runs parallel
- non-preemptive priority
- internal claim race exactly one
- External same ordering
- concurrency case sensitivity

## 9. Maintenance / Recovery

- no busy loop
- earlier deadline wakes
- default max sleep5
- retry due without traffic
- Lease expiry without traffic
- Pause Lease expiry continues / Retry activation stops
- Resume due Retry
- restart overdue Retry/Lease/Retention
- repeated due processing idempotent
- ready pending snapshots survive restart
- running old Runtime -> runner_lost
- completed Run Recovery-only no reopen

## 10. Runner / IPC

Envelope/handshake:

- ready -> start -> terminal -> exit
- malformed UTF8/JSON/protocol/type
- duplicate ready/start/terminal
- terminal+exit matrix

Start:

- persistent_input exact Attempt snapshot
- secret_bindings exact
- secrets key set exact binding names
- binding pointer marker consistency
- Secret missing/invalid before child
- in-memory materialization only
- unbound marker literal not materialized

Runtime Handle:

- request_id correlation exactly one response
- cancel while waiting
- state get/set
- Artifact put/register/materialize

Observation/result:

- log/progress/step validation
- open Step crash incomplete
- ActionFailure propagation
- large result via reserved result file
- path/symlink/size/digest reject
- Validator gets persistent Input only
- stdout/stderr separate/redacted
- old Runner/late message fencing

## 11. Execution Log / Event / Progress

Execution Log:

- internal/external/human/reusable all create one Attempt Log
- normal/debug verbosity
- no automatic full Input/Output body dump
- retention deleted file -> explicit unavailable
- log read path safety

Event:

- state transitions append-only
- progress/log line not duplicated wholesale
- retention audit event no owner FK/event-days exclusion
- `wf_event_list` owner/type/time filters/order/pagination
- event payload Authorization/SecretGuard

Progress:

- Job mode none/explicit/auto
- terminal fraction1 independent conclusion
- internal explicit progress
- External/Human usual0->1
- Reusable parent auto mirrors Child Workflow fraction
- Workflow auto averages concrete Job Runs only
- Dynamic template virtual group not denominator
- generated Job increases denominator and may lower percentage
- mode none returns null
- Progress never affects conclusion

## 12. Retry / Failure

Definition:

- retry absent=max1/disabled
- `retry:{}`=max2/retryable/zero backoff
- numeric/backoff formula
- core retryable map
- ActionFailure/Validator values preserve

Scheduling:

- failed Attempt only
- no new Attempt at schedule/request
- exact pending input/bindings/digest copy
- due executor-specific Attempt creation
- retry condition error preserves original failure

Manual:

- failed Attempt/Input required
- Pre-Attempt failure -> retry_input_unavailable/new Run
- version availability
- run_attempt++
- target with/item/version not re-evaluated
- Secret binding fixed/value may rotate

Descendant:

- blocked/skipped normal reactivation
- failed non-target no implicit Retry
- successful current if re-evaluation
- expected persistent Input re-resolution
- Artifact/version/Payload check
- stored Output Schema/Validator/success_if revalidation
- mismatch -> successful_job_result_not_reusable/new Run

## 13. Dynamic Jobs

Structure:

- Template no Job Run/Attempt
- Root/Nested/3+ depth
- stable/index key
- percent encoding
- no fixed job_key max
- System/Workflow generated limit
- generated count excludes template group
- 1000/1001 atomic rollback
- parent cycle/zero-parent

Ordering:

- source_order/list schema
- asc/desc/default
- invalid types/NaN/Infinity
- stable source tie
- order_rank snapshot

Outcome:

- if=false skipped
- foreach=[] expanded success
- parent zero success/skipped/blocked/cancelled propagation
- mixed skipped/success
- group precedence

Reuse:

- expansion_digest golden
- same expansion exact reuse
- changed source/key/item/order -> dynamic_expansion_not_reusable
- expanded empty -> nonempty rejected in same Run
- whole skipped zero-job group may re-evaluate
- mixed successful group cannot partially re-expand

Persistence:

- root/nested partial unique
- source/digest/outcome recovery

## 14. Reusable Workflow

- registered/relative/path safety
- Binding Definition+Action+Validator versions
- Binding Child effective settings/Retention
- default_runner_pool frozen in Child binding
- System settings change after binding does not alter Retry Child
- Child source_identity inheritance
- Parent/Child state isolation
- parent->child ArtifactRef
- child->parent ArtifactRef
- no Artifact mirror
- Parent auto progress mirrors Child
- Child direct control reject
- cycle/depth/Dynamic/restart duplicate
- Parent run-history upper-bounds child subtree retention

## 15. ArtifactStore / ArtifactRef

- managed put/generation/materialize/path safety
- DB failure orphan cleanup
- canonical ArtifactRef/DB re-resolve
- same-run positive
- cross-run requires persistent Input ref + Authorization
- raw/forged ref reject
- retained/deleted data reject
- cross-run ref does not pin source
- Input ref changes digest
- non-input materialize reuse ineligible
- External no Core fetch/materialize/delete

## 16. External / Human

External:

- Job>Workflow>System lease snapshot
- requeue keeps same config/Attempt
- arbitrary result/spill
- one Task/Attempt/active Lease
- claim race/order
- no heartbeat/renew/transfer
- lease requeue/fail/Pause/Recovery
- stale/cancel submit reject
- submit validation Retry schedules later Attempt
- submit + claim_next + idempotency one transaction
- crash/replay no double next claim
- common Execution Log

Human:

- approve/reject only
- Output null
- first-wins
- Retry new Review later via scheduler
- no timeout/Validator
- no success override/manual skip/rewrite
- common Execution Log

## 17. Persistence schema

Verify **all 18 tables** with SQLite PRAGMA/sqlite_master:

- exact columns
- enum/check invariants
- actual FK vs logical pointer contract
- Dynamic template no Job row
- effective settings exact keys incl default_runner_pool
- ready/pending all-executor invariant
- pending noninternal ready index
- Attempt secret bindings/input digest
- one-running internal partial unique
- Dynamic root/nested partial unique + expansion digest
- Reusable Child settings/Retention columns
- state current/history atomic
- Artifact/log schema
- Runner active slot/liveness
- External Task lease config snapshot + active Lease unique
- Human Review immutable
- idempotency no reserved row/request hash/scope/TTL
- child-first retention
- migration gap/future reject

## 18. Service / MCP / HTTP

Canonical:

- Definition vs Run APIs
- unknown field/Actor injection reject
- pagination/cursors
- Run/Job progress fields
- Output info/read/select
- Task claim no-candidate null
- submit+claim_next exact replay
- Review contracts
- Artifact/Log errors
- `wf_event_list`
- Runner info
- namespaced MCP collision

Forbidden:

- no lease heartbeat/renew/transfer
- no manual Job skip/success override/generic conclusion mutation
- no completed Review rewrite

HTTP:

- exact v1 routes
- workflow_ref query slash support
- Idempotency-Key
- 200/201/400/401/403/404/409/413/500
- no422
- original status/body replay

## 19. Idempotency / Concurrency races

Idempotency:

- request hash excludes request_id/transport only
- Actor/AccessScope isolation in scope
- fast read is advisory only
- BEGIN IMMEDIATE commit-time key/hash recheck
- same hash replay
- different hash conflict
- expired replacement
- no persistent reserved row
- filesystem prepared orphan cleanup
- task_claim exactly one side effect
- task_submit claim_next replay exact next_task

Concurrency:

- same BINARY group race
- queue/reject
- holder release ordering

## 20. Retention

- System all-null unlimited
- Workflow field overrides
- effective policy Run snapshot
- Child binding policy snapshot
- later config change no effect
- completed/non-terminal cutoffs
- all four executor Logs
- normal Event created_at
- Artifact data/metadata owner guard
- Payload follows Run history
- Managed data before metadata
- External data no delete
- Cross-run Artifact no pin
- Child subtree depth-first delete
- system audit survives Run/event deletion
- orphan cleanup audit
- Retention loss makes materialize/reuse fail-closed

## 21. Authorization / Security

- AllowAll/Deny/filtered
- all public read/write
- Event list scope filtering
- Process-local Provider reconstruction
- Action Runner no credential
- Secret binding no value persistence
- Actor/Scope persistence SecretGuard
- log redaction
- managed Artifact byte scan
- cross-run Artifact explicit ref + authorization
- Reusable no scope escalation
- arbitrary shell無し

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

1. `01`〜`12`全受入条件対応表を作成
2. Dependencies/extras Python3.10 Windows/Linux
3. Definition/settings/Registry/reload
4. Persistent Input/Secret binding
5. Integration Bootstrap/IPC
6. Runner Pool/Scheduling
7. Exact 18-table schema/migrations
8. PayloadStore/ArtifactStore
9. Validator/SecretGuard/Authorization
10. External/Human E2E
11. Dynamic virtual group/1000/nested/order/reuse
12. Reusable binding/settings/artifact/progress
13. Retry/Recovery strict reuse
14. Service/MCP/HTTP/Event contracts
15. Idempotency/concurrency/claim_next race tests
16. Common Execution Log/Progress
17. Retention/audit/orphan cleanup

WebUI E2Eは後続。
