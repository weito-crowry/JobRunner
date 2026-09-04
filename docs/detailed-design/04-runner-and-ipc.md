# 04. Runner / IPC 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`, `08-persistence.md`, `09-artifacts-logs-state.md`, `10-retry-recovery-cancel.md`, `12-security-and-secrets.md`

## 1. 目的

Runner Process、Runner Pool、Heartbeat、Common Action Runner、JSON Lines IPC、large result受渡し、Runtime Handle request/response、internal Cancel/Timeout、一時directory、restartを定義する。

## 2. Process構成

```text
Parent System
├─ Runtime / Service
└─ Runner Pool Supervisor
   └─ Runner Process
      ├─ RunnerService -> same SQLite
      ├─ main loop
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
         └─ registered Action
```

MVPではRunner用HTTP/Brokerを必須にしない。Parent/Runnerは同じDB/data root/configを使い、Action Runner childにはDB pathを渡さない。

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

Parent->Runner lifecycle pipe/handleを持ち、Parent消失時はnew claimを止める。旧Runtime更新はfencingで拒否。

## 4. Heartbeat / main-loop liveness

Default:

```text
heartbeat_interval_seconds = 5
runner_lost_after_seconds = 20
main_loop_stale_after_seconds = 15
```

main loopはbounded cycleごとにmonotonic tick更新。Heartbeat Threadはmain loop tickがfreshな場合だけheartbeat更新する。

重いActionは別Processなのでheartbeatを止めない。

## 5. Atomic claim

Runnerは `runtime_instance_id/runner_id/runner_instance_id/pool` でclaim。`03/08`のtransactionでcandidate選択 + Job running + new Attempt + ownershipを確定する。

1 Runnerは同時に1 Job。

## 6. Common Action Runner

```text
bootstrap parent Action Registry
-> action_id + version exact match
-> Action execute
```

Windows spawn前提。Callableをpickle転送しない。

Action:

```python
def action(input_data) -> AnyJson: ...
def action(input_data, runtime) -> AnyJson: ...
```

sync/async対応。ReturnはJSON-compatible value全般。

## 7. IPC v1 transport

Structured channelは専用bidirectional pipe。stdout/stderrとは分離。

```text
UTF-8
1 JSON object / LF
protocol=jobrunner.action-ipc.v1
```

全message:

```text
protocol
type
payload
request_id optional
```

Malformed/unknown protocol/typeは `ipc_protocol_error`。

## 8. Runner -> Action Runner messages

Control:

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

`runtime_response` はChildからのRuntime Handle requestへ対応する。

```json
{
  "type": "runtime_response",
  "request_id": "req_...",
  "payload": {"ok": true, "result": {}}
}
```

Failure:

```json
{"ok": false, "error": {"code": "...", "message": "..."}}
```

## 9. Action Runner -> Runner messages

Async event:

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

Runtime requestは一意 `request_id`必須。Runnerはexactly one `runtime_response`を返す。Childはresponse待ち中もcancel messageを処理できるreader loopを持つ。

## 10. Runtime Handle semantics

### State

```text
state_get(name)
state_set(name,value)
```

current Workflow Run namespaceのみ。`state_set`はSecretGuard + transaction。

### Managed Artifact put

```text
artifact_put_file(name, relative_work_path, media_type?, metadata?)
```

Runnerはpathをcurrent work_dir内へcanonicalizeし、`09`のArtifactStoreへ保存する。成功responseはArtifactRef。

### External Artifact

```text
artifact_register_external(name, uri, ...)
```

CoreはURI fetchしない。

### Artifact materialize

```text
artifact_materialize(artifact_id)
```

Managed Artifactをcurrent work_dir内のCore生成destinationへmaterializeしlocal relative pathを返す。External Referenceはunsupported。

## 11. Large Action Result protocol

Action Returnを巨大なJSON Lines messageとして送らない。

Common Action RunnerはAction return後:

1. JSON-compatible validation
2. reserved result directory `work_dir/.jobrunner/` を作成
3. canonical JSONを `result.json.tmp` へstream/write
4. close後 `result.json` へatomic rename
5. size + SHA-256 digestを計算
6. small `result` IPC messageを送る

```json
{
  "type": "result",
  "payload": {
    "relative_path": ".jobrunner/result.json",
    "size_bytes": 123456,
    "sha256": "..."
  }
}
```

Runnerは:

1. pathがwork_dir内のreserved result pathであることを確認
2. size/digest確認
3. JSON deserialize
4. JSON Schema / `success_if`
5. SecretGuard
6. `08` PayloadStoreへinline/spill保存

これによりOutput sizeに関係なくIPC messageは小さい。

## 12. stdout / stderr / Log

stdout/stderrはRunnerがcaptureしてExecution Logへ。Known Secretはwrite前redact。

Progress:

```text
current >=0
total optional; presentなら >0 and current<=total
```

Open StepはAttempt異常終了時に閉じる。

## 13. Temp / result cleanup

```text
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

Action result fileもtemp配下。PayloadStore commit完了後、Attempt cleanupで削除。

Managed Artifactはput時にdurable ArtifactStoreへcopy済みなのでtemp削除の影響を受けない。

Temp cleanup failureはJob conclusionを変更しない。

## 14. Internal Action completion

Success候補:

- valid result reference
- child exit code 0
- result file size/digest/JSON valid
- Schema / success_if
- SecretGuard
- PayloadStore commit

Failure:

```text
action_exception
action_process_exit
ipc_protocol_error
result_file_invalid
output_validation_failed
secret_value_persistence_blocked
payload_storage_failed
```

## 15. Internal Job timeout

`timeout-minutes`はinternalのみ。未指定無期限。

期限:

1. cancel_requested(reason=timeout)
2. grace default10秒 configurable
3. child未終了ならterminate
4. `job_timeout`
5. Retry policy

## 16. Workflow Cancel vs Parent shutdown

Workflow Cancelはcurrent childへcancelを送りJob conclusion=`cancelled`。

Parent正常shutdownはWorkflow cancelではない。new claim停止後Runner/childをboundedにreapし、Workflow `cancel_requested`を立てない。旧Runtimeの未完了running Attemptは次回起動時に通常 `runner_lost` Recoveryへ渡す。

## 17. Restart policy

Default:

```text
mode=on_failure
max_restarts=5
window_seconds=300
backoff initial1/max30/multiplier2
```

Crash loopは`restart_suppressed`。

## 18. 非目標

CPU/RAM/GPU quota、本格sandbox、arbitrary shellはCore MVPなし。

## 19. 受入条件

1. heartbeat/main-loop stall
2. heavy Action中heartbeat
3. atomic claim
4. Windows spawn/bootstrap
5. stdout/protocol分離
6. Runtime Handle request/response correlation
7. Runtime request待ち中cancel処理
8. managed Artifact put/materialize
9. large JSON result file protocol
10. result path traversal/digest mismatch reject
11. transparent PayloadStore handoff
12. timeout/cancel
13. Parent restart runner_lost
14. old Runner fencing/restart suppression
