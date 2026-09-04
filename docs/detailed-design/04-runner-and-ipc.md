# 04. Runner / IPC 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03-runtime-and-scheduling.md`, `08-persistence.md`, `10-retry-recovery-cancel.md`

## 1. 目的

Runner Process、Runner Pool、Heartbeat、main-loop liveness、Common Action Runner子Process、IPC、internal Job Cancel/Timeout、一時directory、restartを定義する。

## 2. 基本原則

1. Runnerは親起動時に常駐。
2. RunnerはJobをpullしWorkflow Runへ固定しない。
3. Runner管理ProcessとAction実行Processを分離。
4. HeartbeatはRunner管理Processの健全性を示す。
5. Action Runner子ProcessはCore DBへ直接接続しない。
6. Runnerは内部RunnerService/Repository経由で状態更新。
7. IPCはJSON Lines。
8. temp directoryはAttempt終了時削除しsandboxではない。

## 3. Process / Runtime境界

```text
Parent System
├─ Runtime / Service
└─ Runner Pool Supervisor
   └─ Runner Process
      ├─ RunnerService -> same SQLite
      ├─ main loop
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
```

MVPではRunner用HTTP/Brokerを必須にしない。Parent/Runnerは同じDB path/data root/configを使う短命SQLite connectionで内部Serviceを構築する。

Action RunnerにはDB pathを渡さない。

## 4. Runner lifecycle

SupervisorはPool `runner_count`分を起動。Parent->Runner lifecycle pipe/handleを持ち、Parent消失時はEOFを検知してnew claimを止めRunner自身も終了する。

旧RuntimeからのDB更新は `runtime_instance_id` fencingでも拒否。

## 5. Pool default

```text
heartbeat_interval_seconds = 5
runner_lost_after_seconds = 20
main_loop_stale_after_seconds = 15
```

validation:

```text
runner_lost_after_seconds >= heartbeat_interval_seconds * 2
main_loop_stale_after_seconds >= heartbeat_interval_seconds * 2
```

Restart default:

```text
mode=on_failure
max_restarts=5
window_seconds=300
backoff=1s -> max30s, multiplier2
```

## 6. Runner identity/status

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

status:

```text
starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed
```

## 7. main loop / liveness

```text
register -> heartbeat -> idle -> claim -> spawn child -> monitor -> reflect terminal -> cleanup -> idle
```

1 Runnerは同時に1Job。

main loopはbounded cycleごとにmonotonic tick更新。Heartbeat Threadはtickが `main_loop_stale_after_seconds` 内のときだけheartbeat更新する。

Actionは別Processなので長時間Actionでもmain loop/heartbeatを継続できる。

## 8. Atomic claim

`runtime_instance_id/runner_id/runner_instance_id/pool`を渡し、`03/08`のtransactionでcandidate選択 + Job running + new Attempt + Runner ownershipを確定。

競合時は候補再選択/none。

## 9. Common Action Runner

共通entry point:

```text
bootstrap parent Action Registry
-> action_id + version exact match
-> callable execute
```

Windows spawn前提。Callableをpickle転送しない。

Version mismatchは `action_version_mismatch`, retryable=false。

## 10. Action / Runtime Handle

```python
def action(input_data) -> dict: ...
def action(input_data, runtime) -> dict: ...
```

sync/async対応。

Runtime Handle:

```text
log/progress
step_start/step_end
artifact register
state get/set
cancel_requested
work_dir
```

Core変更要求はIPCでRunnerへ送信しRunnerServiceがtransactionを行う。

## 11. IPC v1

```text
UTF-8
1 JSON object / LF line
protocol=jobrunner.action-ipc.v1
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

Malformed/unknown protocol/typeは `ipc_protocol_error`。

## 12. Channel分離

Structured IPCは専用pipe。Action stdout/stderrは別captureしてExecution Logへ。`print()`でprotocolを壊さない。

`start` payload:

- action id/version
- workflow/job/attempt id
- persisted Input
- work_dir
- runtime metadata
- runtime-only materialized Secret

Secretをdebug dumpしない。

## 13. Log / Progress / Step / Artifact / State

Progressは `current>=0`, totalありなら `total>0 && current<=total`。

Step sequenceはRunner側で確定。open StepはAttempt異常終了時に閉じる。

Artifactはmetadata referenceのみ。State操作はcurrent Workflow Run namespaceだけ。

## 14. temp directory

```text
jobrunner-data/runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

内部opaque IDを使いYAML/Dynamic keyをpathへ使わない。

削除失敗はJob conclusionを変更せずwarning/Event。

## 15. Internal Action completion

正常候補:

- result受信
- child exit code 0
- Output validation / success_if通過

failure:

- non-zero exit
- result無し終了
- malformed IPC
- Action exception
- Output validation/success_if failure

## 16. **Internal Job timeout**

本節のtimeoutは `executor: internal` の `timeout-minutes` だけを扱う。

未指定は無期限。

期限到達:

1. `cancel_requested(reason=timeout)`
2. graceful grace default 10秒、configurable
3. child未終了ならAction子Process terminate
4. Attempt `job_timeout` failure
5. Retry policy

Runner Process自体はkillしない。

External/Human/Reusableに `timeout-minutes` は無く、それぞれ `07/06` の規則に従う。

## 17. Workflow Cancel

current internal Jobへcancel request。Workflow cancel由来でActionが協調終了した場合もJob conclusion=`cancelled`。

公開force-kill APIは無し。

## 18. Runner lost / fencing

Heartbeat/liveness消失後、`10`がowned running Attemptを `runner_lost` へ確定。

旧Runtime/Runner/Attemptのlate completionは拒否。

## 19. Graceful Parent shutdown

既定:

1. new claim停止
2. running internal Actionへ **shutdown由来のgraceful cancel request**
3. bounded grace後も残るAction childはterminate
4. Attemptは `cancelled` ではなく `runner_shutdown_interrupted` のretryable failureとして確定できるようにし、次回Runtime起動時に通常Retry/Recovery policyへ渡す
5. Runner stop/reap

Parentの正常shutdownをWorkflow user cancelと混同しない。Workflow `cancel_requested` は立てない。

## 20. 非目標

CPU/RAM/GPU quota、cgroup、Windows Job Object sandboxはMVPなし。任意code隔離は親専用Action内のDocker等。

## 21. 受入条件

1. broker不要same SQLite internal service
2. parent EOF Runner終了
3. heartbeat/main-loop stale
4. heavy Action中heartbeat継続
5. atomic claim
6. spawn/bootstrap/version
7. dedicated IPC + stdout/stderr
8. state/artifact proxy
9. internal timeoutのみ
10. Workflow cancel vs Parent shutdown区別
11. child crash/fencing
12. restart suppression
13. temp cleanup
