# 13. Testing 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `docs/detailed-design/01-workflow-definition.md` 〜 `12-security-and-secrets.md`

## 1. 目的

本書は JobRunner MVP のテスト戦略、レイヤ分割、必須シナリオ、競合・Recovery・Persistence・MCP・Securityの検証方針を定義する。

## 2. 基本原則

1. Runtimeの状態遷移はunit testだけで済ませずintegration testを持つ。
2. SQLite transaction / concurrencyは実DBで検証する。
3. Runner / Action Runnerは実Processを使うE2Eを用意する。
4. External / Human / Retry / Recoveryはnegative caseを必須とする。
5. Definition / Expressionはtable-driven testを多用する。
6. 時刻依存処理はclock abstractionで制御可能にする。
7. random sleepによる不安定な競合testを避け、barrier / hookを用いる。
8. 受入条件は各詳細設計の末尾と本書を対応させる。

## 3. Test layers

```text
Unit
Integration
Process Integration
Adapter Contract
End-to-End
Recovery / Chaos-like
```

## 4. Unit Test

対象例:

```text
YAML parser
Pydantic/typed model validation
CEL/JMESPath wrapper
Input mapping
condition evaluation
retry decision
backoff calculation
priority comparator
dynamic key generation
output aggregation
failure model
```

外部I/Oをmockし、純粋ロジックを高速に回す。

## 5. Integration Test

実SQLite temporary DBを使用する。

対象:

```text
Workflow Run creation
Job activation
atomic claim
Attempt creation
state history
Artifact metadata
Event append
Dynamic Job expansion
External lease
Human review
Idempotency
Retention metadata
```

## 6. Process Integration Test

実Runner Process + Common Action Runner Processを起動する。

テストAction Registryを用意する。

代表Action:

```text
success_action
failure_action
sleep_action
progress_action
step_action
stdout_action
stderr_action
artifact_action
cancel_aware_action
crash_action
hang_action
```

## 7. Adapter Contract Test

同じService operationに対し、Python API / MCP / Web Adapterで意味が変わらないことを検証する。

WebUI画面そのものは別設計だが、HTTP Adapter contractはService testを再利用する。

## 8. End-to-End Test

最低限以下のWorkflowを実際に流す。

### 8.1 Basic

```text
A -> B -> C
```

Output mapping / needs / logs / Workflow conclusionを確認。

### 8.2 Branch

```text
A
├-> B
└-> C
    ↓
    D
```

ただし1 Workflow Run内internal Jobは同時実行最大1であることを確認。

### 8.3 External

```text
internal -> external_llm -> internal
```

### 8.4 Human

```text
internal -> human -> internal
```

### 8.5 Dynamic

```text
generate -> foreach[N] -> aggregate
```

### 8.6 Reusable

```text
parent -> child workflow -> parent continuation
```

## 9. Definition Validation Test

positive / negative tableを用意する。

negative例:

```text
unknown key
duplicate Job ID
missing needs target
DAG cycle
unknown Action
unknown Runner Pool
action + uses conflict
invalid executor
invalid retry
invalid timeout
invalid foreach
invalid concurrency
invalid expression
```

Workflow Run rowが作られる前に失敗するケースを確認する。

## 10. Expression Test

CEL:

```text
boolean
number
string
list/map access
helper functions
success/failure/cancelled/always
missing value
null
wrong type
```

JMESPath:

```text
selection
filter
projection
invalid syntax
unexpected result type
```

`${{ ... }}`全体式と文字列埋め込みを分けて検証する。

## 11. Input / Output Test

```text
Workflow input required/default/type
Job with mapping
$base + override
needs outputs
Artifact refs
state refs
Secret refs
Output JSON validation
JSON Schema success/failure
NaN/Infinity rejection
```

RetryでInputが変化しないことを必須検証。

## 12. Scheduling Test

```text
Workflow priority
Job priority
order_by
ready time tie-break
pool routing
unknown pool rejection
1 Run internal Job max1
multiple Workflow Runs parallel
pause no-new-claim
resume
priority update no preemption
```

複数Runnerを実Processまたはthreaded test harnessで競合させる。

## 13. Atomic Claim Test

同じqueued Jobに複数Runnerが同時claim要求。

期待:

```text
success count = 1
attempt count = 1
owner runner = exactly one
```

少なくとも数十〜数百回繰り返すstress testを別カテゴリで用意する。

## 14. Runner / IPC Test

```text
start handshake
ready event
log/progress/step/result/error
malformed JSON
unknown event type
protocol version mismatch
stdout/stderr capture
Action exception
child process non-zero exit
Action process crash
```

Protocol errorでCore stateが不整合にならないことを確認。

## 15. Heartbeat Test

fake clockを使用する。

```text
normal heartbeat
late heartbeat within grace
lost threshold exceeded
runner_lost transition
restart policy
max_restarts
restart window
backoff
restart suppressed
```

Actionが長時間実行中でもHeartbeat継続を確認。

## 16. Timeout Test

```text
no timeout -> long action success
timeout -> cancel requested
graceful exit
forced child termination
failure code job_timeout
retry after timeout
```

実時間を長く待たないようclock/process hookを使う。

## 17. Dynamic Job Test

必須:

```text
0 items
1 item
many items
stable key
index fallback
duplicate key
1000 jobs allowed
1001 jobs rejected
system override
workflow override
nested foreach
arbitrary depth representative case
atomic rollback
order_by
output aggregation
artifact aggregation
runtime restart after expansion
```

上限超過時に部分Jobが残らないことを確認。

## 18. Reusable Workflow Test

```text
parent-child success
input mapping
output mapping
artifact mapping
child failure propagation
child cancel propagation
retry -> new child run
direct cycle
indirect cycle
nested child
state isolation
actor/scope inheritance
runtime restart relation recovery
```

## 19. External LLM Test

```text
claim success
concurrent claim exactly one
lease expiry
stale lease submit rejection
cancelled task submit rejection
result schema validation
claim_next
retry -> new task
restart with valid lease
restart with expired lease
idempotent submit
```

## 20. Human Review Test

```text
approve
reject
concurrent review first wins
cancel waiting_review
late review rejection
retry after reject
idempotent review_submit
actor Event
```

## 21. Retry Test

```text
manual retry
auto retry disabled default
max attempts
retry condition false
retry condition true
backoff
retry exhausted
Input frozen
old output not current
new successful artifact current
```

## 22. Recovery Test

Runtimeを意図的に停止し再起動する。

ケース:

```text
queued Job
running internal Job
waiting_external valid lease
waiting_external expired lease
waiting_review
paused Run
child Workflow running
dynamic expansion completed
retry backoff waiting
```

Recovery後に重複Job / Attempt / Child Runが生成されないことを確認。

## 23. Fencing Test

旧Runner instanceからlate completionを送信。

期待:

```text
rejected
current Attempt unchanged
Event/diagnostic available
```

旧runtime_instance_idも同様。

## 24. Persistence Test

```text
fresh migration
incremental migration
migration rollback/failure
foreign key
WAL behavior
busy timeout
transaction rollback
JSON serialization
Definition YAML snapshot
Definition hash
```

Migration fixtureは全versionからlatestへのupgrade pathを可能な範囲で保持する。

## 25. Workflow State Test

```text
get absent
set first revision
set overwrite
history old/new
last-write-wins
restart persistence
step/job/attempt attribution
Child state isolation
```

## 26. Artifact / Log Test

```text
Artifact metadata register
same-name generations
current resolution
history
log append
stdout/stderr
log tail/offset
large log not embedded in wf_info
Step boundary
Temp directory deletion
Temp delete failure warning
```

## 27. Authorization / Secrets Test

```text
AllowAll
Deny provider
filtered AccessScope
all state-changing ops call authorize
Secret runtime injection
Secret absent from DB
Secret absent from Event
Secret absent from Definition snapshot resolved values
redaction hook
log path traversal
Artifact URI no auto-fetch
```

## 28. Idempotency Test

各対象operationで:

```text
same key + same request -> same result
same key + different request -> conflict
concurrent same key -> single side effect
```

対象:

```text
start
pause
resume
cancel
retry
task_submit
review_submit
```

## 29. Failure Injection

test hook経由で以下を注入できると望ましい。

```text
DB write failure
process spawn failure
log write failure
temp cleanup failure
heartbeat stop
Action process kill
IPC malformed frame
Artifact registration failure
```

本番コードへ広範なtest-only分岐を埋め込まずdependency injectionで行う。

## 30. Property / Fuzz Test候補

MVP必須ではないが以下は有効。

```text
YAML parser invalid structures
Expression nested data
Dynamic key generation
state transition illegal paths
IPC frame decoder
```

## 31. Performance smoke test

大規模分散性能は目標外だが、退行検出として以下を測る。

```text
1000 Dynamic Jobs expansion
1000 Job metadata read
large Event history pagination
large Execution Log tail
3 heavy Runner pool scheduling simulation
```

閾値は実装後baseline化する。

## 32. Platform Test

少なくとも:

```text
Windows 11
Linux
```

Process spawn / signal / path / filesystem差異を確認。

Windowsを第一級対象とする。

## 33. Python version

Packageが宣言するsupported Python versionsすべてでCIを回す。

CEL/JMESPath等依存library互換性をCIで固定する。

## 34. CI構成

推奨job:

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform matrix
```

重いstress testは通常PRとnightly相当で分離してよい。

## 35. Test data isolation

各testはtemporary DB / temporary data rootを使用。

固定の開発DBを共有しない。

Process test終了時に子Processを必ずreapする。

## 36. Determinism

以下を注入可能にする。

```text
Clock
ID generator
Random source optional
Process launcher
```

テストで再現可能なID/時刻を使えるようにする。

## 37. Coverage方針

line coverageの数値だけを完了条件にしない。

重要state transition / conflict / recovery / negative pathが網羅されていることを優先する。

特に以下は必須。

```text
atomicity
idempotency
fencing
cancel
retry
recovery
Dynamic Job rollback
Secret non-persistence
```

## 38. MVP completion gate

MVP実装完了条件:

1. 01〜12の各受入条件に対応testがある
2. 全migration test pass
3. Windows/Linux process integration pass
4. concurrent claim test pass
5. restart recovery test pass
6. External/Human E2E pass
7. Dynamic Job 1000件 test pass
8. Reusable Workflow nested/cycle test pass
9. Secret non-persistence test pass
10. adapter contract test pass

## 39. 非目標

MVPで以下は必須としない。

```text
Kubernetes E2E
multi-machine chaos test
browser UI E2E
massive distributed load test
security penetration test automation
```

WebUI E2EはWebUI詳細設計作成後に別途追加する。
