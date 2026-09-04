# 04. Runner / IPC 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/01-workflow-definition.md`
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/03-runtime-and-scheduling.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner における Runner Process、Runner Pool、Heartbeat、Common Action Runner Process、Runner と Action Runner 間の IPC、Job / Attempt の実行開始・終了・異常終了・Cancel・Timeout・一時作業ディレクトリの扱いを定義する。

対象:

- Runner Process の常駐方式
- Runner Pool の起動・停止
- Runner identity
- Runner heartbeat
- Runner lost 判定
- Job claim 後の実行フロー
- Common Action Runner Process
- Action Registry bootstrap
- Runner ↔ Action Runner IPC
- stdout / stderr / Execution Log
- Progress / Step / Artifact 登録イベント
- Cancel
- Job timeout
- Action Process 異常終了
- Runner Process 異常終了
- Temporary working directory
- Runner restart policy
- 親システム停止時の graceful shutdown

Retry / Recovery policy の最終判定は `10-retry-recovery-cancel.md`、Execution Log の保存構造は `09-artifacts-logs-state.md` で定義する。

## 2. 基本原則

1. Runner は親システム起動時に自動起動し、常駐する。
2. Runner は Workflow Run に固定しない。
3. Runner が取得する実行単位は Job とする。
4. Runner は Job を pull する。
5. Runner 管理 Process と実際の Action 実行 Process を分離する。
6. Heartbeat は Runner Process 自体の生存確認に使う。
7. Action が長時間実行中でも Heartbeat は継続する。
8. Action Process の障害と Runner Process の障害を区別する。
9. Action Process は Core DB を直接更新しない。
10. Core への状態反映は Runner を経由する。
11. Runner と Action Runner の IPC はローカル JSON Lines を基本とする。
12. Job / Attempt ごとに専用 temporary working directory を作る。
13. temporary working directory は Attempt 終了時に削除する。
14. 永続化が必要なファイルは Action / 親システム側の責任で保存する。
15. temporary working directory は security sandbox ではない。

## 3. Process 構成

MVP の標準構成は以下。

```text
Parent System Process
├─ JobRunner Runtime / Service
├─ Action Registry bootstrap provider
└─ Runner Pool Supervisor
   ├─ Runner Process A
   │  ├─ Heartbeat Thread
   │  └─ Common Action Runner Process
   │     └─ Action
   ├─ Runner Process B
   │  ├─ Heartbeat Thread
   │  └─ Common Action Runner Process
   │     └─ Action
   └─ ...
```

Runner Process は 1 Job を実行中のみ Action Runner Process を 1 個持つ。

1 Runner が同時に複数 Job を実行しない。

## 4. Runner Pool

### 4.1 登録

Runner Pool は親システム側で事前登録する。

例:

```python
runner_pools = {
    "default": RunnerPoolConfig(runner_count=2),
    "heavy": RunnerPoolConfig(runner_count=3),
}
```

Workflow YAML の `runs-on` は登録済み Pool 名を参照する。

未登録 Pool は Workflow Run 開始前 validation error。

### 4.2 MVP 設定

Runner Pool は少なくとも以下を持つ。

```text
name
runner_count
restart_policy
heartbeat_interval_seconds
runner_lost_after_seconds
```

Heartbeat 設定は System default を継承できる。

MVP では以下を行わない。

- CPU quota
- RAM quota
- GPU quota
- cgroup
- Windows Job Object による resource enforcement
- Pool ごとの Action allow-list

### 4.3 起動

親システムの Runtime 初期化後、Runner Pool Supervisor が `runner_count` 分の Runner Process を起動する。

Runner 起動失敗時は restart policy に従う。

親システム自体の起動を失敗させるか degraded 起動を許すかは System policy とし、Core default は以下とする。

- `default` Pool が 0 Runner で起動不能: Runtime degraded
- Service / MCP / Web から状態確認可能
- Workflow 定義自体の読み込みは可能
- internal Job は実行不可

## 5. Runner identity

Runner Process ごとに少なくとも以下を持つ。

```text
runner_id
runner_instance_id
runtime_instance_id
pool_name
pid
started_at
last_heartbeat_at
status
```

意味:

- `runner_id`: Pool 内の論理 Runner 名。再起動後も同じ slot を表せる
- `runner_instance_id`: Process 起動ごとに新規発行する一意 ID
- `runtime_instance_id`: 親 Runtime 起動ごとの ID

同じ `runner_id` が再起動しても `runner_instance_id` は変わる。

古い `runtime_instance_id` / `runner_instance_id` からの更新は拒否する。

## 6. Runner status

Runner の主要 status:

```text
starting
idle
claiming
running
stopping
stopped
lost
restart_suppressed
```

`running` の場合は現在の以下を保持する。

```text
workflow_run_id
job_run_id
attempt_id
action_runner_pid
```

## 7. Runner main loop

Runner の基本ループ:

```text
start
↓
register runner instance
↓
start heartbeat thread
↓
idle
↓
claim executable Job
↓
if none:
    wait / wake-up
    ↓
    retry claim
↓
prepare Attempt
↓
start Common Action Runner Process
↓
monitor Action Runner
↓
collect result / error
↓
complete Attempt
↓
cleanup temporary directory
↓
idle
```

Runner 自身が Workflow DAG を評価しない。

実行可能 Job の選択は Runtime Service / Scheduler 側へ委譲する。

## 8. Atomic Job claim

Runner は Pool 名と自身の identity を指定して Job を claim する。

概念 API:

```text
claim_next_job(
  runtime_instance_id,
  runner_id,
  runner_instance_id,
  pool_name
)
```

Runtime 側は 1 transaction 内で以下を行う。

1. Runner identity が current であることを確認
2. Pool が一致することを確認
3. 実行可能 Job を選択
4. 同一 Workflow Run に別 internal Job が running でないことを確認
5. Job / Attempt を running へ遷移
6. `runner_id` / `runner_instance_id` を記録
7. claim 結果を返す

競合で claim に失敗した場合、Runner は error 扱いにせず再取得する。

## 9. Heartbeat

### 9.1 目的

Heartbeat は Runner Process の生存確認に使用する。

Action 本体の進捗確認には使用しない。

### 9.2 実装

Runner Process 内で Heartbeat 専用 Thread を持つ。

```text
Runner Process
├─ main loop
└─ Heartbeat Thread
```

Heartbeat Thread は Action 実行中も継続する。

### 9.3 既定値

MVP default:

```text
heartbeat_interval_seconds = 5
runner_lost_after_seconds = 20
```

System / Pool 設定で変更可能。

`runner_lost_after_seconds` は `heartbeat_interval_seconds` より十分大きくなければならない。

最低 validation:

```text
runner_lost_after_seconds >= heartbeat_interval_seconds * 2
```

### 9.4 Heartbeat payload

概念上、少なくとも以下を送る。

```json
{
  "runtime_instance_id": "rt_...",
  "runner_id": "heavy-2",
  "runner_instance_id": "ri_...",
  "pool": "heavy",
  "status": "running",
  "job_run_id": "job_...",
  "attempt_id": "att_...",
  "pid": 12345,
  "at": "..."
}
```

### 9.5 Runner lost 判定

最終 Heartbeat から `runner_lost_after_seconds` を超えた Runner は lost candidate とする。

lost 確定処理は transaction 内で current runner identity を再確認する。

すでに新 `runner_instance_id` が同じ slot を引き継いでいる場合、古い Runner を理由に current Job を壊してはならない。

Runner lost 後の Attempt / Job Recovery は `10-retry-recovery-cancel.md` に従う。

## 10. Common Action Runner Process

### 10.1 目的

Runner は Action ごとに直接専用 script を起動しない。

共通の Action Runner entry point を起動する。

```text
Runner
↓
Common Action Runner Process
↓
Action Registry bootstrap
↓
action_id / version 解決
↓
Action callable 実行
```

### 10.2 起動情報

Runner は Action Runner Process へ少なくとも以下を渡す。

```text
action_id
action_version
workflow_run_id
job_run_id
attempt_id
job_input
work_dir
required runtime metadata
secret references / injected secret values
```

### 10.3 Registry bootstrap

Action Runner Process は親システムが提供する共通 bootstrap を呼び、Action Registry を再構築する。

Registry の callable 自体を Parent Process から pickle 等で転送しない。

Windows の spawn を前提としても成立する方式とする。

### 10.4 Action version 照合

Action Runner Process は Registry 再構築後、要求された `action_id + version` が存在することを確認する。

不一致:

```text
category = validation_error
code = action_version_mismatch
retryable = false
```

として fail-closed。

## 11. Action callable

正式対応する形:

```python
def action(input_data) -> dict:
    ...
```

または Runtime Handle 付き:

```python
def action(input_data, runtime) -> dict:
    ...
```

async も正式対応する。

Action Runner が sync / async の差を吸収する。

Action が返す通常 Output は JSON-compatible でなければならない。

## 12. IPC 方針

### 12.1 基本

Runner ↔ Common Action Runner 間は JSON Lines ベースのローカル IPC とする。

1 message = 1 JSON object + LF。

Protocol encoding:

```text
UTF-8
LF line ending
JSON object only
```

JSON array / scalar 単体 message は禁止。

### 12.2 双方向通信

Runner → Action Runner:

- `start`
- `cancel_requested`
- 将来拡張用 control message

Action Runner → Runner:

- `ready`
- `log`
- `progress`
- `step_started`
- `step_finished`
- `artifact_registered`
- `result`
- `error`
- `exiting`

### 12.3 Protocol version

すべての structured message は protocol version を持つ。

例:

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "progress",
  "payload": {
    "current": 10,
    "total": 100,
    "message": "loading"
  }
}
```

未知 protocol version は受理しない。

未知 `type` は protocol error とする。

## 13. Runner → Action Runner message

### 13.1 start

Runner 起動直後に 1 回だけ送る。

概念 payload:

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "start",
  "payload": {
    "action_id": "fx.run_backtest",
    "action_version": "3",
    "workflow_run_id": "wr_123",
    "job_run_id": "jr_456",
    "attempt_id": "att_2",
    "input": {},
    "work_dir": "...",
    "runtime": {},
    "secrets": {}
  }
}
```

Secret 値を Debug Log へ出してはならない。

### 13.2 cancel_requested

Workflow / Job Cancel を Runner が検知した場合に送る。

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "cancel_requested",
  "payload": {
    "reason": "workflow_cancelled"
  }
}
```

送信は idempotent。

複数回送られても Action Runner は問題なく扱う。

## 14. Action Runner → Runner message

### 14.1 ready

bootstrap、Action 解決、start payload の基本検証が完了した時点で送る。

### 14.2 log

概念 payload:

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "log",
  "payload": {
    "level": "info",
    "message": "loaded 100 rows",
    "step": "load_data"
  }
}
```

`level` MVP:

```text
debug
info
warning
error
```

### 14.3 progress

```json
{
  "type": "progress",
  "payload": {
    "current": 30,
    "total": 100,
    "message": "evaluating"
  }
}
```

条件:

```text
current >= 0
total > 0
current <= total
```

`total` 不明の indeterminate progress は将来拡張とし、MVP では `current/total` を基本とする。

### 14.4 step_started

```json
{
  "type": "step_started",
  "payload": {
    "step_id": "load_data",
    "name": "Load data"
  }
}
```

### 14.5 step_finished

```json
{
  "type": "step_finished",
  "payload": {
    "step_id": "load_data",
    "conclusion": "success"
  }
}
```

MVP の Step conclusion:

```text
success
failure
cancelled
```

Step 自体は Scheduler / Retry 単位ではない。

### 14.6 artifact_registered

Action が親側で永続化した Artifact を Core へ登録する要求。

```json
{
  "type": "artifact_registered",
  "payload": {
    "name": "dataset",
    "uri": "parent://...",
    "media_type": "application/parquet",
    "size": 12345,
    "digest": "..."
  }
}
```

Artifact 実体は IPC で送らない。

### 14.7 result

成功時、terminal message として 1 回だけ送る。

```json
{
  "type": "result",
  "payload": {
    "output": {}
  }
}
```

`result` 後に追加の状態変更 message を送ってはならない。

### 14.8 error

失敗時、terminal message として 1 回だけ送る。

```json
{
  "type": "error",
  "payload": {
    "category": "execution_error",
    "code": "action_exception",
    "message": "...",
    "retryable": false,
    "details": {}
  }
}
```

### 14.9 exiting

Action Runner が正常な終了処理へ入ったことを知らせる optional message。

`result` / `error` が terminal truth であり、`exiting` は診断補助。

## 15. stdout / stderr

Action の通常 `print()` が structured IPC を壊してはならない。

そのため、MVP では以下を原則とする。

- structured IPC: 専用 pipe / file descriptor / multiprocessing connection 等の dedicated channel
- Action stdout: Action Runner が capture
- Action stderr: Action Runner が capture
- capture 内容: `log` event として Runner へ転送

OS / Python 実装都合で dedicated channel の実装方式は変更可能だが、structured protocol と user stdout/stderr を論理的に分離することは必須。

stdout / stderr の非常に長い 1 行については、Runner 側で安全な上限を設け、Execution Log へ分割記録可能とする。

## 16. IPC validation

Runner は Action Runner から受け取った message を毎回検証する。

少なくとも:

- valid UTF-8
- valid JSON
- object type
- protocol version
- known message type
- required fields
- payload type
- Attempt identity 一致
- terminal message 重複禁止

Protocol 違反時:

```text
category = protocol_error
code = invalid_action_ipc
```

とし、Action Process を停止対象とする。

## 17. Runtime Handle

Action Runner は高度な Action に Runtime Handle を提供する。

概念 API:

```python
runtime.log(...)
runtime.progress(...)
runtime.step_start(...)
runtime.step_end(...)
runtime.artifact_register(...)
runtime.state_get(...)
runtime.state_set(...)
runtime.cancel_requested()
```

Runtime Handle は Action が Core DB を直接操作する API ではない。

必要な要求を IPC へ変換し Runner / Runtime Service 経由で処理する。

state get/set の exact request/response IPC は `09-artifacts-logs-state.md` で定義する。

## 18. Temporary working directory

### 18.1 Path

概念 path:

```text
jobrunner-data/
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_number>/
```

実際の filesystem-safe name は内部 ID を使用する。

YAML Job ID をそのまま path element として信頼しない。

### 18.2 作成

Attempt を Runner が claim した後、Action Runner 起動前に作成する。

作成失敗は execution error。

### 18.3 Action への提供

Action は Runtime Handle または start context から `work_dir` を取得できる。

Action が current working directory に依存することは推奨しない。

必要に応じて Common Action Runner が process cwd を `work_dir` に設定してよいが、canonical access は明示 path とする。

### 18.4 削除

Attempt の terminal 確定後、原則必ず削除する。

対象:

- success
- failure
- cancelled
- timeout

削除失敗は Attempt conclusion を変更しない。

代わりに warning Event / Log を残し、Retention / maintenance で再削除可能にする。

### 18.5 永続化責任

残す必要がある file / directory は Action が Attempt terminal 前に親システム側へ保存する。

Core は temporary directory の内容を自動 Artifact 化しない。

## 19. Action Process lifecycle

### 19.1 起動

Runner は Common Action Runner Process を child process として起動する。

OS ごとに、親 Runtime 全体を巻き込まず子 Process 単独を停止できる起動方式を使う。

Windows では new process group 相当、POSIX では new session / process group 相当を利用可能とする。

### 19.2 正常終了

正常 success:

1. `result` 受信
2. Output validation
3. Action Runner exit code 0 確認
4. Attempt success 確定

正常 failure:

1. `error` 受信
2. Action Runner 終了
3. Attempt failure 確定

### 19.3 terminal message なしで exit 0

Protocol error とする。

```text
code = missing_terminal_message
```

### 19.4 terminal message なしで non-zero exit

Execution error とする。

```text
code = action_process_exit
```

exit code / stderr tail 等を `details` へ保存可能。

### 19.5 `result` 後に non-zero exit

成功と見なさない。

`result` 送信後でも Process が異常終了した場合は protocol / execution integrity failure とする。

Attempt success は terminal message と正常 Process exit の両方を確認してから確定する。

## 20. Job timeout

Job timeout は YAML で指定された場合のみ有効。

未指定なら無期限。

Timeout 計測開始:

```text
Attempt が running へ遷移した時点
```

Timeout 到達時:

1. Runner が `cancel_requested` を Action Runner へ送る
2. grace period を待つ
3. 終了しない場合は Action Runner Process を停止
4. Attempt を timeout failure とする

MVP の timeout grace period は System 設定とし、既定 10 秒程度を推奨値とするが、exact default は実装時設定ファイルで固定する。

`timeout` は `runner_lost` と区別する。

## 21. Cancel

### 21.1 Workflow / Job Cancel 検知

Runner は実行中 Attempt について cancel request を確認する。

検知経路は以下を組み合わせてよい。

- Runtime からの local wake-up
- Runner polling
- Heartbeat response に control 情報を含める

実装方式は 1 つに固定しないが、Cancel 反映が Heartbeat interval より大幅に遅延しないことを目標とする。

### 21.2 Graceful Cancel

Runner は Action Runner へ `cancel_requested` を送る。

Action は `runtime.cancel_requested()` で安全な停止点を確認できる。

Action が終了した場合、Attempt conclusion は `cancelled`。

### 21.3 Cancel に応答しない Action

通常の Cancel API 自体は force kill を意味しない。

ただし親システム shutdown、Job timeout、Runner recovery 等、Process を残せない運用局面では child Process を停止できる。

Cancel の grace / force policy の最終仕様は `10-retry-recovery-cancel.md` に従う。

## 22. Runner Process 異常終了

Runner Process が死ぬと Heartbeat が止まる。

Runtime は `runner_lost_after_seconds` 経過後に lost を検知する。

その Runner が実行していた Attempt は `runner_lost` recovery 対象。

Action Runner child Process が orphan になる可能性があるため、Runner 起動時 / Runtime recovery 時に、known child PID / process group を可能な範囲で回収する。

ただし OS 全体の任意 orphan process 探索を Core の必須責務にはしない。

## 23. Action Process 異常終了

Runner が生存している状態で Action Runner が異常終了した場合、Runner が即時検知する。

Heartbeat timeout を待たない。

Attempt failure:

```text
category = execution_error
code = action_process_exit
```

Retryable 判定は Retry / Recovery policy に委譲する。

## 24. Runner restart policy

Runner Pool ごとに設定可能。

MVP minimum:

```text
mode: on_failure | never
max_restarts
window_seconds
backoff_initial_seconds
backoff_max_seconds
```

`on_failure`:

- unexpected exit のみ再起動
- parent shutdown に伴う正常停止は再起動しない

Crash loop 防止:

- `window_seconds` 内で `max_restarts` を超えた場合 `restart_suppressed`
- 自動再起動を止める
- Event を記録
- Service / MCP / Web から確認可能

## 25. Graceful shutdown

親システム終了時:

1. Runtime を stopping にする
2. 新しい Job claim を停止
3. idle Runner へ stop 指示
4. running Runner は現在 Job の扱いを shutdown policy に従う
5. Heartbeat Thread 停止
6. Runner Process 終了

MVP default shutdown policy:

- running Action へ cancel request
- configurable grace period 待機
- grace 超過後は child Action Process を停止
- Attempt は `runner_lost` ではなく shutdown/cancel 系 failure または cancelled として記録する

親システムの通常 shutdown と crash recovery を区別する。

## 26. Security boundary

Runner / Action Runner の Process 分離は安定性・監視性のための分離であり、security sandbox ではない。

Action は親システムと同じ OS user 権限で動作し得る。

MVP では以下を保証しない。

- filesystem isolation
- network isolation
- syscall isolation
- memory / CPU quota
- untrusted code execution safety

任意 code 実行が必要な親システムは専用 Action を実装し、必要なら Docker 等の sandbox / container を Action 側で利用する。

## 27. Observability

Runner / Action Runner について最低限以下を Event / Log へ記録する。

- Runner started
- Runner idle
- Job claimed
- Action Runner started
- Action Runner PID
- Action Runner ready
- Action result / error
- Action process exit code
- Cancel requested
- Timeout
- Runner lost
- Runner restarted
- Restart suppressed
- Temporary directory cleanup failure
- Protocol error

## 28. Idempotency / 重複防止

Runner から Runtime への状態更新は Attempt identity を必須にする。

古い Attempt / 古い Runner instance からの late update は拒否する。

例:

- Attempt 1 timeout 後に古い Action Runner が `result` を送信
- Runner 再起動後に旧 Runner instance が heartbeat
- Retry Attempt 2 開始後に Attempt 1 の artifact event

これらは current state に適用しない。

必要に応じて diagnostics Event だけ記録する。

## 29. 実装上の推奨

MVP の Python 実装では以下を優先する。

- `multiprocessing` / `subprocess` を OS 差異を意識して薄くラップする
- Windows spawn 前提で Registry bootstrap を再実行可能にする
- IPC message は typed model で validation
- Runner main loop と Heartbeat Thread の共有状態は最小化
- DB transaction は Runner Process ではなく Runtime / persistence service に集約
- child Process stdout/stderr capture は deadlock しないよう専用 reader thread / async reader 等を使う
- Secret 値の redaction を logging pipeline の入口で適用

## 30. 受入条件

MVP 実装は少なくとも以下を自動テストする。

1. Runner Pool の指定数 Runner が起動する
2. Runner が Job を pull して 1 件だけ claim する
3. 2 Runner が同じ Job を同時 claim できない
4. 同一 Workflow Run の internal Job が 2 件同時に running にならない
5. Action Runner Process が別 Process として起動する
6. Registry bootstrap から Action が解決される
7. Action version mismatch が fail-closed になる
8. sync Action が実行できる
9. async Action が実行できる
10. stdout / stderr が structured IPC を壊さない
11. `log` / `progress` / Step event を受信できる
12. Artifact 登録 event を受信できる
13. success `result` が Output として保存される
14. Action exception が structured failure になる
15. non-zero exit が failure になる
16. terminal message なし exit 0 が protocol error になる
17. malformed JSON message が protocol error になる
18. Attempt ごとの temporary directory が作成される
19. success 後に temporary directory が削除される
20. failure 後にも temporary directory が削除される
21. cleanup failure が Attempt conclusion を上書きしない
22. Heartbeat が Action 実行中も継続する
23. Runner Process kill 後に heartbeat timeout で lost になる
24. Action Runner Process kill は Runner が即時検知する
25. Job timeout 未設定時は長時間 Action を自動 timeout しない
26. Job timeout 設定時は規定時間後に timeout failure になる
27. Cancel request が Action Runner へ通知される
28. 古い Runner instance の heartbeat が拒否される
29. 古い Attempt の late result が拒否される
30. restart policy が crash loop を抑止できる
31. 親システム graceful shutdown 時に新しい Job claim が停止する
32. Runner が Workflow Run に固定されず、次 Job を別 Runner が取得できる

## 31. 今後の拡張余地

MVP 後の候補:

- Heartbeat に CPU / RSS diagnostics を追加
- Action Runner の resource limit
- Docker / container Runner backend
- remote Runner
- GPU label / capability routing
- heartbeat lease token
- bidirectional streaming IPC の別 transport
- structured tracing / OpenTelemetry
- indeterminate progress

これらを追加しても、`Runner が Job を pullし、Common Action Runner Process で Action を実行する` という基本 contract は維持する。
