# 04. Runner / IPC 詳細設計

- Status: Draft v0.7
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02`, `03`, `08`, `09`, `10`, `12`

## 1. 目的

Runner Process、Runner Pool、Heartbeat、Supervisor liveness、Common Action Runner、JSON Lines IPC、structured Action failure、large result受渡し、Runtime Handle、一時directory、restartを定義する。

## 2. Process構成

```text
Parent System
├─ Runtime / Service
└─ Runner Pool Supervisor
   └─ Runner Process
      ├─ RunnerService -> same SQLite
      ├─ Validator Registry bootstrap
      ├─ main loop
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
         └─ Action Registry bootstrap -> registered Action
```

MVPではRunner用HTTP/Broker必須無し。Parent/Runnerは同DB/data root/config。Action Runner childへDB pathを渡さない。

## 3. Runner lifecycle / identity

Identity:

```text
runner_id
runner_instance_id
runtime_instance_id
pool_name
pid
```

Status:

```text
starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed
```

Parent->Runner lifecycle pipe/handleを持つ。Parent消失時new claim停止。旧Runtime更新はfencing reject。

## 4. Heartbeat / main-loop liveness

Default:

```text
heartbeat_interval_seconds = 5
runner_lost_after_seconds = 20
main_loop_stale_after_seconds = 15
```

main loopはbounded cycleでmonotonic tick更新。Heartbeat Threadはmain loop tick fresh時だけheartbeat更新。

重いActionは別Processなのでheartbeatを止めない。

### 4.1 Supervisor liveness scan

Runner Pool Supervisorは:

- child Process exitをOS process handleで即時検知
- 生存Processは `heartbeat_interval_seconds` 以下の周期でDB heartbeatをscan
- `now - last_heartbeat_at > runner_lost_after_seconds` かつ current `runner_instance_id`一致ならRunnerをlost候補にする
- current Attempt ownershipを再確認して`10`の`runner_lost` Recoveryを1回だけ起動
- 新instanceへ同じrunner_idが移った後の旧heartbeatを理由にcurrent Attemptを壊さない

Heartbeat scanはRuntimeのWorkflow SchedulerではなくRunner supervision責務。

## 5. Atomic claim

Runnerは `runtime_instance_id/runner_id/runner_instance_id/pool` でclaim。`03/08` transactionでcandidate + Job running + new Attempt + ownership。

1 Runner同時1 Job。

## 6. Common Action Runner

```text
bootstrap Action Registry
-> action_id + version exact match
-> Action execute
```

Windows spawn前提。Callableをpickle転送しない。

Action:

```python
def action(input_data) -> AnyJson: ...
def action(input_data, runtime) -> AnyJson: ...
```

sync/async対応。

### 6.1 Structured `ActionFailure`

```python
raise ActionFailure(
    code="rate_limited",
    message="upstream temporarily unavailable",
    retryable=True,
    details={"provider": "example"},
)
```

Contract:

```text
code: non-empty string
message: string
retryable: boolean default false
details: JSON-compatible optional
category = action
```

通常未処理exception=`action_exception`, retryable=false。

## 7. IPC v1 transport

Dedicated bidirectional pipe。stdout/stderr分離。

```text
UTF-8
1 JSON object / LF
protocol=jobrunner.action-ipc.v1
```

全message=`protocol/type/payload/request_id optional`。

Malformed/unknown protocol/type -> `ipc_protocol_error`。

## 8. Runner -> Action Runner messages

```text
start
cancel_requested
runtime_response
```

`start`:

- action id/version
- workflow/job/attempt id
- persisted Input
- work_dir
- runtime metadata
- runtime-only materialized Secrets

`runtime_response` はRuntime Handle requestへexact request_idで返す。

## 9. Action Runner -> Runner messages

Async:

```text
ready
log
progress
step_started
step_finished
result
error
exiting
```

Runtime Handle request:

```text
state_get
state_set
artifact_put_file
artifact_register_external
artifact_materialize
```

Requestは一意`request_id`必須。Runnerはexactly one response。Childはresponse待ち中もcancel受信可能。

## 10. Runtime Handle

State:

```text
state_get(name)
state_set(name,value)
```

Managed Artifact:

```text
artifact_put_file(name, relative_work_path,...)
artifact_materialize(artifact_ref)
```

External Artifact:

```text
artifact_register_external(name, uri,...)
```

State/Artifact変更はRunnerService transaction。Path規則・ArtifactRef shape・cross-run authorizationは`09`。

### 10.1 `artifact_materialize`

Runnerは受け取ったArtifactRefを`artifact_id`でDB再解決し、caller supplied metadataを信頼しない。

許可:

- same Workflow Run owned Managed Artifact、または
- different Workflow Runだがcanonical ArtifactRefがcurrent persistent Job Input内に明示存在し、AuthorizationProviderがsource Artifact readを許可

拒否:

- raw/forged cross-run artifact_idのみ
- deleted/retained-away Managed data
- External Reference Artifact
- current work_dir外destination

Runtime中にpersistent Input外Artifactをmaterializeした場合はResult Reuse eligibility=falseとして記録する。

## 11. Large Action Result protocol

Action Return本体をJSON Linesへ載せない。

Common Action Runner:

1. JSON-compatible validation
2. `work_dir/.jobrunner/result.json.tmp` write
3. close -> `result.json` atomic rename
4. size/SHA-256
5. small result IPC

Runner:

1. reserved path内確認
2. size/digest
3. JSON deserialize
4. optional JSON Schema
5. optional Custom Validator (`01`) via Runner-side Validator Registry
6. optional `success_if`
7. SecretGuard
8. PayloadStore

Validatorにはpersistent Inputを渡し、materialized Secret valueを渡さない。

## 12. stdout / stderr / Log

stdout/stderrはExecution Log。Known Secret write前redact。

Progress `current>=0`, total optional; presentなら`>0`かつ`current<=total`。

Open StepはAttempt異常終了時に閉じる。

## 13. Temp / cleanup

```text
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

Result fileもtemp配下。PayloadStore commit後cleanup。

Managed Artifactはput時durable copy済み。

Cleanup failureはJob conclusion変更なし。

## 14. Internal completion

Success候補:

- valid result reference / exit0
- JSON/Schema
- Custom Validator valid
- success_if true
- SecretGuard
- PayloadStore commit

Failure例:

```text
action_exception
<parent ActionFailure code>
validator_exception
domain_validation_failed
ipc_protocol_error
result_file_invalid
output_validation_failed
secret_value_persistence_blocked
payload_storage_failed
artifact_access_forbidden
artifact_data_unavailable
```

## 15. Internal Job timeout

`timeout-minutes` internalのみ。未指定無期限。

1. timeout cancel
2. grace default10秒 configurable
3. child terminate
4. `job_timeout`
5. Retry policy

## 16. Workflow Cancel vs Parent shutdown

Workflow Cancelはchildへcancel、Job cancelled。

Parent正常shutdownはWorkflow cancelではない。new claim停止、bounded reap、cancel_requestedは立てない。未完了running Attemptは次回起動でrunner_lost Recovery。

## 17. Restart policy

Default:

```text
mode=on_failure
max_restarts=5
window_seconds=300
backoff initial1/max30/multiplier2
```

Crash loop -> restart_suppressed。

## 18. 非目標

CPU/RAM/GPU quota、本格sandbox、arbitrary shellはCore MVP無し。

## 19. 受入条件

1. heartbeat/main-loop stall
2. Supervisor heartbeat scan/lost exactly once
3. heavy Action中heartbeat
4. atomic claim
5. Windows spawn/bootstrap
6. ActionFailure structured propagation
7. stdout/protocol分離
8. Runtime Handle correlation/cancel
9. same-run Artifact materialize
10. explicit cross-run Input Artifact materialize + authorization
11. raw cross-run ID / External materialize reject
12. large JSON result file
13. Custom Validator internal path
14. result path/digest reject
15. timeout/cancel
16. Parent restart runner_lost
17. old Runner fencing/restart suppression
