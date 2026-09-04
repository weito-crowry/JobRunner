# 04. Runner / IPC 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03-runtime-and-scheduling.md`, `08-persistence.md`, `10-retry-recovery-cancel.md`

## 1. 目的

Runner Process、Runner Pool、Heartbeat、main-loop liveness、Common Action Runner子Process、IPC、Cancel/Timeout、一時directory、restartを定義する。

## 2. 基本原則

1. Runnerは親システム起動時に常駐起動。
2. RunnerはJobをpullしWorkflow Runへ固定しない。
3. Runner管理ProcessとAction実行Processを分離する。
4. HeartbeatはRunner管理Processの健全性を示し、Actionの進捗を示さない。
5. Action Runner子ProcessはCore DBへ直接アクセスしない。
6. Runnerは内部RunnerService/Repository経由でDB状態を更新する。
7. Runner↔Action Runner IPCはJSON Lines。
8. Attempt専用temp directoryは終了時削除。sandboxではない。

## 3. Process構成とRuntime境界

```text
Parent System Process
├─ JobRunner Runtime / Service
└─ Runner Pool Supervisor
   ├─ Runner Process
   │  ├─ RunnerService / Repository -> same SQLite DB
   │  ├─ main loop
   │  ├─ Heartbeat Thread
   │  └─ Common Action Runner Process
   └─ ...
```

**MVPではRunner用のHTTP/Socket Brokerを必須にしない。** Parent ProcessとRunner Processは同じDB path/data root/configを使い、それぞれ短命SQLite connectionで内部Service/Repositoryを構築する。

RunnerServiceが担当する内部operation:

```text
runner_register
runner_heartbeat
claim_next_job
attempt_complete / fail
artifact/state/event reflection
runner_stop
```

SQLite transaction/policyはCore service/repositoryへ集約し、Runner main loopへSQLを散在させない。

Action Runner子ProcessにはDB path/credentialを渡さない。

## 4. Supervisor / Runner lifecycle

親起動後、Poolごとの`runner_count`分を起動する。親終了時はgraceful stopを送る。

Supervisor->Runnerには専用lifecycle pipe/handleを持たせる。親Process消失でpipe EOF/handle closeを検知したRunnerは新Jobをclaimせず自分も終了する。Runtime再起動後に旧RunnerがDB更新し続けることは`runtime_instance_id` fencingでも拒否する。

## 5. Runner Pool config

```text
name
runner_count
restart_policy
heartbeat_interval_seconds
runner_lost_after_seconds
main_loop_stale_after_seconds
```

System default:

```text
heartbeat_interval_seconds = 5
runner_lost_after_seconds = 20
main_loop_stale_after_seconds = 15
```

最低validation:

```text
runner_lost_after_seconds >= heartbeat_interval_seconds * 2
main_loop_stale_after_seconds >= heartbeat_interval_seconds * 2
```

`runs-on`の解決は`01`。

## 6. Runner identity/status

```text
runner_id                 # Pool内論理slot
runner_instance_id        # Process起動ごとに新規
runtime_instance_id       # Parent Runtime起動ごと
pool_name
pid
started_at
last_heartbeat_at
status
```

status:

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

running時はcurrent `job_run_id/attempt_id/action_runner_pid`を保持する。

## 7. Runner main loop

```text
register -> heartbeat start -> idle
   -> claim
   -> prepare temp/log
   -> spawn Common Action Runner
   -> monitor child / cancel / timeout
   -> terminal resultをRunnerServiceへ反映
   -> cleanup temp
   -> idle
```

1 Runnerは同時に1 Jobだけ実行する。

### 7.1 main-loop tick

Heartbeat Threadだけが生存し、main loopが固まる偽陽性を防ぐ。

- main loopはidle wait、claim前後、child monitor loopの各bounded cycleでmonotonic `main_loop_tick`を更新する。
- child monitorは無期限blocking waitを使わず、bounded poll/readを行う。
- Heartbeat Threadは `now - main_loop_tick <= main_loop_stale_after_seconds` のときだけheartbeatを更新する。
- stale超過時、Heartbeat Threadはheartbeat更新を停止し、Supervisor診断用状態を残してRunner Process終了を要求する。
- Actionは別Processなので、重いActionがmain tickを止める理由にはならない。

これにより「Actionが長い」と「Runner main loopが死んだ」を分離する。

## 8. Atomic claim

Runnerは`runtime_instance_id/runner_id/runner_instance_id/pool`を渡す。Runtimeは`03/08`のtransactionでJobを選択し、新Attemptを作る。

競合は通常errorではなく候補再選択/none。

## 9. Heartbeat

payload:

```text
runtime_instance_id
runner_id
runner_instance_id
pool
status
job_run_id optional
attempt_id optional
pid
at
main_loop_tick_age
```

lost判定時はtransaction内でcurrent instanceを再確認する。新instanceへ同じrunner_idが移っている場合、旧instanceを理由にcurrent Jobを壊さない。

## 10. Common Action Runner

RunnerはActionごとのscriptを直接起動せず、共通entry pointを起動する。

```text
Common Action Runner
 -> parent-provided Registry bootstrap
 -> action_id + version exact match
 -> callable execute
```

Windows spawnでも成立させる。Callableをpickleで親から送らない。

Version mismatch:

```text
category=validation
code=action_version_mismatch
retryable=false
```

## 11. Action callable / Runtime Handle

```python
def action(input_data) -> dict: ...
def action(input_data, runtime) -> dict: ...
```

sync/async両対応。

Runtime Handle:

```text
log
progress
step_start / step_end
artifact register
state get/set
cancel_requested
work_dir
```

Runtime HandleはAction Runner内proxyで、state/artifact等のCore変更要求をIPCでRunnerへ送り、RunnerServiceがtransactionを行う。

## 12. Action IPC v1

Encoding:

```text
UTF-8
1 JSON object / line
LF
protocol = jobrunner.action-ipc.v1
```

Runner -> child:

```text
start
cancel_requested
```

Child -> Runner:

```text
ready
log
progress
step_started
step_finished
artifact_registered
state_get
state_set
result
error
exiting
```

未知protocol/type、malformed JSON、required field欠落は`ipc_protocol_error`。

## 13. IPC channel分離

Actionの通常stdout/stderrをprotocol transportとして使わない。

- structured control/data: dedicated pipe/file descriptor/OS pipe
- stdout/stderr: RunnerがcaptureしExecution Logへ

これによりActionの`print()`がJSON Linesを壊さない。

## 14. `start`

Runnerは子起動後に1回だけ送る。

含む:

```text
action_id/version
workflow_run_id/job_run_id/attempt_id
persisted input
work_dir
runtime metadata
runtime-only materialized secrets
```

Secretをdebug dumpしない。

## 15. Log / Progress / Step

Log level: `debug/info/warning/error`。

Progress:

```text
current >= 0
total optional; presentなら >0 and current<=total
message/unit optional
```

StepはsequenceをRunner側で確定し、open StepはAttempt異常終了時にfailure/incompleteへ閉じる。

## 16. Artifact / State IPC

Artifact登録要求にはmetadata referenceだけを含む。実体はAction/親側保存済みであること。

State get/setはRunnerServiceを通しcurrent Workflow Run namespaceだけ操作する。Child Workflowは自身のRun stateのみ。

## 17. temp directory

filesystem-safe内部IDを使う。

```text
jobrunner-data/runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

Attempt開始時作成、success/failure/cancelを問わず終了時削除。削除失敗はJob conclusionを書き換えずwarning/Eventを残す。

## 18. Process monitoring / normal completion

正常result受信 + child exit code 0 + output validation成功でAttemptをsuccess候補として確定する。

以下はfailure:

- child非zero exit
- result無し終了
- malformed IPC
- Action exception
- output validation/success_if failure

failureはstructured failureとして`10`へ渡す。

## 19. Timeout

Job timeout未指定は無期限。

期限到達:

1. `cancel_requested(reason=timeout)`送信
2. configurable graceful grace（System default 10秒）
3. child未終了ならRunnerがchild Processをterminate
4. `job_timeout` failure
5. Retry policy

Runner Process自体は殺さない。

## 20. Workflow Cancel

Runnerはcurrent Jobへのcancel requestを検知し、子へ送る。協調停止したActionも最終conclusionはWorkflow cancel由来なら`cancelled`。

公開force-kill APIはMVPに置かない。

## 21. Runner lost / late update

Runner heartbeat消失後、`10`がowned running Attemptを`runner_lost`へ確定する。

旧Runtime/旧runner instance/旧Attemptからのlate completionはfencingして拒否する。

## 22. Restart policy

System default:

```text
mode = on_failure
max_restarts = 5
window_seconds = 300
backoff.initial_seconds = 1
backoff.max_seconds = 30
backoff.multiplier = 2
```

- `never`: 自動再起動なし
- `on_failure`: unexpected exitのみ再起動
- window内上限超過 -> `restart_suppressed`
- 正常parent shutdownはrestart countへ含めない

全値Pool/Systemで変更可能。

## 23. Graceful parent shutdown

1. new claim停止
2. running childへ通常は継続猶予/parent policyによりcancel
3. Runner stop request
4. child/Runner reap
5. runner status stopped

MVPの既定はparent shutdown時にrunning Actionへgraceful cancelし、Parent再起動後はRecoveryする。無限待機しない。

## 24. Resource/Sandbox非目標

MVPはCPU/RAM/GPU quota、cgroup、Windows Job Object sandboxを提供しない。任意コード実行が必要なら親側専用Action内でDocker等を利用する。

## 25. 受入条件

1. Parent+複数Runner常駐
2. mandatory broker不要 / same SQLite internal service
3. parent lifecycle pipe EOFで旧Runner終了
4. heartbeat正常/lost
5. main loop staleでheartbeat停止
6. heavy child中もmain tick/heartbeat継続
7. atomic claim
8. spawn/bootstrap/version check
9. JSON Lines dedicated channel + stdout/stderr分離
10. state/artifact proxy
11. timeout/cancel
12. child crash
13. old Runner fencing
14. restart defaults/suppression
15. temp cleanup
