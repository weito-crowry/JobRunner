# 13. Testing 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`〜`12-security-and-secrets.md`

## 1. 目的

JobRunner MVPのunit/integration/process/E2E/recovery/security/adapter contractテストを定義し、設計上のatomicity/idempotency/fencingを実DB・実Processで検証する。

## 2. 原則

1. 状態遷移はunitだけで済ませずSQLite integrationを持つ。
2. Runner/Action Runnerは実Process testを持つ。
3. Clock/ID/Process launcherを注入可能にしrandom sleep依存を避ける。
4. Concurrent testはbarrier/hookを使う。
5. 各詳細設計の受入条件を少なくとも1 testへtraceする。
6. Windows 11とLinuxをfirst-class対象とする。

## 3. Test layers

```text
Unit
SQLite Integration
Process Integration
Adapter Contract
End-to-End
Recovery / Fault Injection
```

## 4. Definition validation

Positive/negative table:

```text
valid Workflow
unknown top-level/Job/settings field
duplicate YAML mapping key
YAML merge key <<
custom/object tag
duplicate Job ID
missing needs / DAG cycle
invalid executor field combination
internal explicit/default/unregistered runs-on
external/human/reusable action/runs-on conflict
Workflow top-level outputs
invalid success_if/retry/timeout/external config
Reusable uses URL/absolute/path traversal
```

Invalid startではWorkflow Run rowが作られないこと。

## 5. Expression/Input/Output

CEL:

```text
boolean strictness
number/string/list/map
missing vs null
invalid context
declared needs外参照拒否
```

JMESPath:

```text
filter/projection
syntax/evaluation failure
foreach unexpected type
```

Contexts:

```text
inputs/env/state
needs normal
needs Dynamic group jobs/outputs/artifacts full key
item/iteration.current/parent/ancestors
failure/outputs
Workflow output jobs context
```

Helpers:

```text
success/failure/cancelled/always
continue-on-error effective success
upstream skipped -> default skipped
upstream failure -> default blocked
explicit if false -> skipped
cancel -> cancelled/no new activation
```

`continue-on-error`はactivation時snapshotでRetry時変化しない。

Job Output:

```text
JSON/schema
NaN/Infinity reject
4 MiB exactly allowed
4 MiB+1 reject output_too_large
```

## 6. Secret tests

```text
internal Action with Secret success
external_llm/human/reusable/condition Secret ref reject
persistent Inputにreferenceのみ
Secret value absent DB/Event/Log/idempotency
per-Attempt materialize
Secret rotation後Retryはnew value使用
missing Secret child spawn前failure
redaction hook
```

## 7. Scheduling / status

```text
Workflow queued/running/paused/completed
Job queued/running/waiting_external/waiting_review/waiting_child/completed
Job paused status不存在
Workflow priority > Job priority > order_by > source > ready_at
1 Workflow Run internal max1
multiple Workflow Runs parallel
Pause no new claim/activation
Resume
Cancel no-new-activation
```

## 8. Atomic internal claim

複数Runnerを同じcandidateへbarrier同期。

期待:

```text
claim success exactly 1
Attempt created exactly 1
Runner owner exactly 1
partial unique index violationが外へ漏れずconflict/retry処理
```

数十〜数百回stressカテゴリも用意。

## 9. Runner process / liveness / IPC

実Processで:

```text
parent supervisor -> runner persistent
mandatory HTTP/Broker無しでsame SQLite internal service
parent lifecycle pipe EOF -> old Runner exit
heartbeat 5s/lost 20s fake clock
main loop tick normal
main loop artificial hang -> heartbeat stops/lost
heavy Action child -> main tick/heartbeat継続
Registry bootstrap/version mismatch
JSON Lines dedicated channel
Action print/stdout/stderrがprotocolを壊さない
malformed/unknown protocol/type
state/artifact Runtime Handle proxy
child exception/nonzero/no-result/hang
```

## 10. Timeout / Cancel Runner

```text
no timeout long Action success
timeout cancel request
grace period
child terminate -> job_timeout
Workflow cancel cooperative Action
late Runner completion fencing
```

## 11. Dynamic Jobs

Root:

```text
0/1/many
stable key/index fallback
special chars percent encoding
raw key 256-byte boundary
full job_key 2048-byte boundary
```

Nested:

```text
foreach.parent/items
parent_a condition[x] vs parent_b condition[x] no collision
iteration parent/ancestors exact snapshot
3+ nested levels representative
```

Atomic/limit:

```text
1000 generated allowed
1001 rejected with 0 partial rows
DB failure mid expansion rollback
restart after committed expansion no duplicate
```

Ordering/group:

```text
order_by string/number
null/bool/object/mixed type reject
source tie
full-key needs group status/conclusion
full-key output/artifact lookup
```

Dynamic + Reusable/External/HumanもE2E。

## 12. Reusable Workflow

```text
registered ID/local ./path
URL/absolute/../ traversal rejection
binding created first activation
Child source changes after binding
Parent Retry -> same binding/new Child Run
Action version mismatch after binding
Workflow top-level jobs output mapping
state isolation
Actor/Scope inheritance
cycle direct/indirect
nested reusable
Parent cancel propagation
Parent pause does not pause started Child
public Child pause/resume/cancel/retry/priority reject
Child info/log read allowed with auth
one binding per parent Job
one Child per parent Attempt
runtime restart relation recovery
```

## 13. External LLM

```text
activation -> exactly one Attempt/Task
concurrent claim exactly one
active lease unique index
requeue expiry -> same Attempt/new Lease
fail expiry -> failure/retry policy
valid/stale/expired/cancel submit
submit Output+Artifact atomic
claim_next failure does not rollback submit
Pause blocks new claim but existing submit allowed
Pause does not freeze expires_at
Retry -> new Attempt/Task
restart valid/expired lease
idempotent claim/submit
```

## 14. Human Review

```text
activation -> one pending Review, outcome null
approve/reject
concurrent submit first wins
cancel pending/late submit reject
Pause retains pending and submit allowed
Retry -> new Attempt/Review
unique Attempt Review constraint
idempotency/actor Event
```

## 15. Retry

Automatic:

```text
disabled default
retryable default matrix
max-attempts
if true/false/error
backoff/retry_not_before
scheduled retry creates no Attempt
executor activation creates next Attempt
```

Manual:

```text
Job-only failed requirement
non-terminal failed Job retry
completed failed Run reopen
run_attempt increment
Run conclusion/failure/output/completed_at cleared
old Attempts preserved
blocked descendants reset
successful/skipped/cancelled/other failed descendants not reset
Dynamic expansion not regenerated
idempotent concurrent same request
recovery never reopens terminal Run
```

## 16. Recovery/fencing

Force Runtime/Runner stop at controlled points:

```text
queued
running internal before/after child result
waiting_external valid/expired lease
waiting_review
waiting_child
paused
concurrency wait
Dynamic expansion before/after commit
retry backoff
```

Expect no duplicate Attempt/Task/Review/Child/Expansion。

Old runtime/runner late heartbeat/completion rejected。

## 17. Persistence/migration

```text
fresh migration
all prior schema version -> latest
migration failure fail-closed
WAL/busy_timeout
FK/check constraints
UUID4 prefix format
internal running partial unique
Dynamic root/nested expansion unique
Reusable binding/Child unique
External Task/active Lease unique
Human pending/outcome constraint
state current/history atomic
Concurrency BEGIN IMMEDIATE race
Idempotency same transaction/TTL expiry
safe relative execution_log path
```

## 18. Artifact / Log / State

```text
Artifact immutable generations/current successful Attempt
failed Attempt Artifact not current
Dynamic full-key Artifact map
External Artifact submit
Log append/flush/tail/offset
large log absent from run info
internal-ID log/temp paths
path traversal rejection
Step normal/crash close
Progress known/indeterminate/Dynamic denominator
state get/set/history/last-write-wins/restart
Child state isolation
temp cleanup/failure warning
retention Event
```

## 19. Service/Adapter contract

同じtyped Service operationをPython/MCP/Web Adapterで意味一致させる。

```text
wf_definition_list/info
wf_start
wf_run_list/info
wf_pause/resume/cancel/retry/priority_update
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_runner_info
```

MCP:

```text
namespace validation/collision
parent tool subset
```

`wf_info`のようにDefinition/Runが曖昧な旧operationを公開しない。

## 20. Authorization

全public read/write operationがProviderを通ることをspy Providerで検証。

```text
AllowAll
Deny
filtered scope
Definition/Run/Review/Artifact/Log/Runner read
state-changing write
Child read vs direct control
```

## 21. Idempotency

対象operationごと:

```text
same key + same request -> same result
same key + different request -> conflict
concurrent same key -> one side effect
DB fault between side effect/resultを作れないsame transaction設計
TTL 24h内 replay
expiry後record retention/replay非保証
```

## 22. Failure injection

Dependency injection/hookで:

```text
DB write/commit failure
process spawn failure
IPC malformed
child kill
heartbeat/main loop tick stop
log write/temp cleanup failure
Artifact registration failure
```

本番ロジックに広範なtest-only分岐を入れない。

## 23. Performance smoke

分散性能目標ではないが退行検知:

```text
1000 Dynamic expansion
1000 Job metadata read
Event cursor pagination
large log tail
3 Runner scheduling simulation
```

実装後baseline化。

## 24. Platform/CI

最低:

```text
Windows 11
Linux
supported Python versions (MVP >=3.10)
```

CI例:

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform matrix
```

Stressは通常PRと分離可能。

## 25. Determinism/Test isolation

Injectable:

```text
Clock
ID generator
Process launcher
Random optional
```

各testはtemp DB/data root。Process testは全childをreap。

## 26. MVP completion gate

1. 01〜12の受入条件に対応test
2. all migration tests
3. Windows/Linux process integration
4. concurrent claim/concurrency/idempotency race
5. restart recovery/fencing
6. External/Human E2E
7. Dynamic 1000 + nested identity
8. Reusable binding/cycle/direct control
9. Secret non-persistence
10. Adapter contract
11. Manual Retry reopen semantics

WebUI browser E2EはWebUI詳細設計後に追加する。
