# 04. Runner / IPC 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `12`

## 1. 目的

Runner Process、Runner Pool、Process-local Integration Bootstrap、Heartbeat、Supervisor liveness、Common Action Runner、JSON Lines IPC v1、structured Action failure、large result受渡し、Runtime Handle、一時directory、restartを実装可能な正規契約として定義する。

## 2. Process構成

```text
Parent System
├─ Runtime / Service
└─ Runner Pool Supervisor
   └─ Runner Process × runner_count
      ├─ Integration Bootstrap(role=runner)
      ├─ RunnerService -> same SQLite
      ├─ Validator/Auth/Secrets/Store
      ├─ main loop
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
         └─ Integration Bootstrap(role=action_runner)
```

MVPではRunner用HTTP/Broker必須無し。Action Runner childへDB path/provider/storage credentialを渡さない。

## 3. Parent Integration Bootstrap

親システムはimport可能entrypointを1つ設定する。

例:

```text
myapp.workflow_integration:configure_jobrunner
```

Conceptual signature:

```python
def configure_jobrunner(builder, role: str) -> None: ...
```

Role=`parent|runner|action_runner`。

Builder:

```text
register_action(action_id, version, callable, metadata?)
register_validator(validator_id, version, callable)
set_authorization_provider_factory(factory)
set_secrets_provider_factory(factory)
set_payload_store_factory(factory) optional
set_artifact_store_factory(factory) optional
```

Role境界:

- parent: Registry/Auth/Secrets/Store factory/public Service
- runner: Registry metadata+Validator/Auth/Secrets/Store/RunnerService
- action_runner: Action callable解決だけ必須

Action RunnerへDB/Auth/Secrets/Store credentialを自動公開しない。

Parent/Runner/Action Runner bootstrapのsnapshot version不一致は`action_version_mismatch|validator_version_mismatch`でfail-closed。

Runtime hot registrationを主要運用にしない。Runner recycle/restartでbootstrapを再実行する。

## 4. Runner Pool

Canonical config:

```text
name non-empty
runner_count integer>=1
restart_policy.mode on_failure|never
restart_policy.max_restarts integer>=0
restart_policy.window_seconds finite>0
restart backoff initial>=0,max>=initial,multiplier>=1 finite
heartbeat_interval_seconds finite>0
runner_lost_after_seconds finite>0
main_loop_stale_after_seconds finite>0
```

Defaults:

```text
runner_count parent-defined
restart mode=on_failure
max_restarts=5
window=300s
backoff=1..30 x2
heartbeat=5s
lost=20s
main-loop-stale=15s
```

Validation:

```text
lost >= heartbeat*2
stale >= heartbeat*2
```

Poolはrouting/Runner数/restart-livenessだけを決める。Action allow-list、resource quota、sandbox、Pool global pause無し。

## 5. Runner identity/lifecycle

```text
runner_id                 # Pool内logical slot
runner_instance_id        # Process起動ごと
runtime_instance_id       # Parent Runtime起動ごと
pool_name
pid
```

Status:

```text
starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed
```

Parent lifecycle pipe EOFでnew claim停止。旧Runtime/旧instance updateはfencing reject。

## 6. Heartbeat / main-loop liveness

Main loopはidle/claim/child monitorのbounded cycleごとにmonotonic tick更新。

Heartbeat Threadはmain-loop tick fresh時のみheartbeat更新。重いActionは別Processなのでheartbeatを止めない。

Supervisor:

- OS process exit即時検知
- heartbeat interval以下の周期でscan
- `now-last_heartbeat > lost_after` + current instance一致でlost候補
- Attempt ownership再確認後`runner_lost` Recovery exactly once
- stale old instanceでnew instanceのJobを壊さない

## 7. Atomic claim

Runnerは`runtime_instance_id/runner_id/runner_instance_id/pool`でclaim。

`03/08`の1 transactionで:

- eligible Job selection
- same Run running internal無し
- Job running
- new Attempt
- Runner ownership/current Job

を確定。

1 Runner同時1 Job。Resolved `runs-on` exact一致のみclaim。

## 8. Common Action Runner callable

Action:

```python
def action(input_data) -> AnyJson: ...
def action(input_data, runtime) -> AnyJson: ...
```

sync/async対応。

Structured failure:

```python
raise ActionFailure(
    code="rate_limited",
    message="upstream temporarily unavailable",
    retryable=True,
    details={"provider": "example"},
)
```

`code` non-empty、message string、retryable bool default false、details JSON-compatible optional。

Unhandled exception=`action_exception`, retryable=false。

## 9. IPC transport v1

Dedicated bidirectional byte pipe/socketpair-equivalent local transport。stdout/stderrとは分離。

Wire:

```text
UTF-8
one JSON object per line
LF terminator
protocol = "jobrunner.action-ipc.v1"
```

Canonical envelope:

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "...",
  "request_id": null,
  "payload": {}
}
```

Rules:

- `protocol/type/payload` required
- `payload` object required
- `request_id` required only Runtime Handle request/response
- unknown top-level field reject
- unknown message type reject
- malformed UTF-8/JSON -> `ipc_protocol_error`
- message order is pipe order; separate sequence field無し
- Output本体はIPCへ送らず§17 result file protocol

## 10. Handshake / lifecycle order

正規順序:

1. Runner spawns Common Action Runner
2. Child executes `Integration Bootstrap(role=action_runner)`
3. Child sends exactly one `ready`
4. Runner validates child still current owner
5. Runner sends exactly one `start`
6. Child resolves exact Action ID/version and executes
7. Child sends zero or more async/runtime request messages
8. Child sends exactly one terminal `result` **or** `error`
9. Child sends optional `exiting`
10. OS process exits

`ready`前のchild->Runner message、`start`重複、terminal message重複は`ipc_protocol_error`。

Bootstrap自体が失敗して`ready`を送れずchild exitした場合=`action_process_exit`またはbootstrap-specific diagnostic。Attemptはfailure。

## 11. Child -> Runner `ready`

```json
{
  "type": "ready",
  "payload": {
    "pid": 1234,
    "protocol_version": 1
  }
}
```

`protocol_version=1` exact。

## 12. Runner -> Child `start`

Payload:

```text
action_id: non-empty string
action_version: non-empty string
workflow_run_id
job_run_id
attempt_id
persistent_input: object
work_dir: string path
secrets: object<string,string>
runtime:
  cancel_requested: boolean
```

`persistent_input`にはSecret reference markerが残る。`secrets`には当該Attemptで必要なname->materialized valueだけを入れる。

Common Action RunnerはAction呼出直前にreference markerを`secrets`からmemory上で解決した**execution input**を作る。Execution input/Secret値をfile/log/debug dumpへ保存しない。

Unknown/missing SecretはRunnerがstart前に検出するため通常childを起動しない。

`work_dir`はAttempt専用absolute pathだが、childが外部へ返すpathは相対pathのみ。

## 13. Runner -> Child `cancel_requested`

Payload:

```text
reason = workflow_cancel|timeout
requested_at RFC3339 UTC
```

Child reader loopはAction実行中も受信しRuntime Handleのcancel flagを更新する。

- workflow_cancel: cooperative終了しても最終Job conclusionはcancelled
- timeout: grace後未終了ならRunner terminate、failure=`job_timeout`

Parent normal shutdownはWorkflow cancel messageとして扱わない。

## 14. Runtime Handle request/response

Child->Runner requestは一意non-empty `request_id`必須。

Runnerは各requestへexactly one:

```json
{
  "type": "runtime_response",
  "request_id": "req_...",
  "payload": {
    "ok": true,
    "result": {}
  }
}
```

または:

```json
{
  "type": "runtime_response",
  "request_id": "req_...",
  "payload": {
    "ok": false,
    "error": {
      "code": "...",
      "message": "...",
      "retryable": false,
      "details": null
    }
  }
}
```

Runtime request outstanding中もChild reader loopはcancelを処理する。

Terminal `result|error`送信後のnew Runtime requestは禁止。

### 14.1 `state_get`

Request payload:

```text
name: non-empty string
```

Success result:

```text
found: boolean
value: any JSON value optional when found
revision: integer optional when found
```

`state_get`成功利用はAttempt `reuse_eligible=false`を記録。

### 14.2 `state_set`

Request:

```text
name: non-empty string
value: any JSON-compatible value
```

Response:

```text
revision: integer>=1
```

RunnerServiceがSecretGuard + current/history transaction。

### 14.3 `artifact_put_file`

Request:

```text
name: non-empty string
relative_work_path: non-empty relative path
media_type: string optional
metadata: object optional
```

Response=`09` canonical ArtifactRef。

### 14.4 `artifact_register_external`

Request:

```text
name: non-empty string
uri: non-empty string
media_type optional
size_bytes integer>=0 optional
digest optional
metadata object optional
```

Response=External canonical ArtifactRef。Core fetch無し。

### 14.5 `artifact_materialize`

Request:

```text
artifact_ref: canonical ArtifactRef object
```

Response:

```text
relative_work_path: Core-generated relative path
```

Runner DB re-resolve + `09/12` scope/Authorization。External Artifact reject。

Persistent Input外Artifact materializeはreuse_eligible=false。

## 15. Child async observation messages

### 15.1 `log`

```text
level = debug|info|warning|error
message: string
```

Runner受信時刻をlog timestampとし、child自由timestampをSource of Truthにしない。

### 15.2 `progress`

```text
current: finite number >=0
total: finite number >0 optional
message: string optional
unit: string optional
```

`total`ありなら`current<=total`。

### 15.3 `step_started`

```text
step_key: non-empty string unique within Attempt
name: non-empty string
metadata: object optional
```

Runnerが受信順で`sequence`を採番しDB Step IDを生成。ChildはDB step_idを生成しない。

### 15.4 `step_finished`

```text
step_key: existing open step key
conclusion = success|failure|cancelled
metadata: object optional
```

Unknown/already-closed step_keyはprotocol error。Open StepはAttempt異常終了時に`incomplete`でCoreが閉じる。

## 16. Child terminal `error`

PayloadはStructured Failure:

```text
category: non-empty string
code: non-empty string
message: string
retryable: boolean
details: JSON-compatible optional
```

Common Action Runner mapping:

- ActionFailure -> category=action + parent fields
- unhandled exception -> category=action, code=action_exception, retryable=false
- action version mismatch -> validation/action_version_mismatch/false

Childは`error`後Action処理を終了し、原則OS exit code non-zero。

Runnerはerror payloadをSecretGuardしてAttempt failureへ反映する。Error message/detailsにSecretが含まれる場合は安全なreplacement failureへ変換しSecretを保存しない。

## 17. Child terminal `result` / large result file

Action return本体をJSON Linesへ載せない。

Child:

1. Action returnがJSON-compatibleか確認
2. `work_dir/.jobrunner/` Core reserved dir
3. `result.json.tmp`へcanonical-json-v1 write
4. fsync/close相当後`result.json`へatomic rename
5. size + SHA-256
6. `result`送信

Payload:

```text
relative_path = ".jobrunner/result.json"
size_bytes: integer>=0
sha256: lowercase hex string
```

Runner:

1. exact reserved relative path確認
2. symlink/path escape拒否
3. file existence/size/digest
4. deserialize JSON
5. optional JSON Schema
6. optional Validator
7. optional success_if
8. SecretGuard
9. PayloadStore commit

## 18. `exiting` / OS exit

Optional `exiting` payload:

```text
reason = result|error
```

`exiting`はterminal outcomeのSource of Truthではない。Source of Truthは`result|error` + OS exit + Runner validation。

Outcome:

- result + exit0 + validation成功 -> success候補
- result + nonzero exit -> `action_process_exit` failure; result非採用
- error + nonzero exit -> error failure採用
- error + exit0 -> protocol/runtime failureとしてerror failure採用しdiagnostic warning
- no terminal + exit nonzero -> `action_process_exit`
- no terminal + exit0 -> `action_result_missing`
- result/error両方 -> `ipc_protocol_error`

## 19. stdout / stderr

Action child stdout/stderrはRunnerが別pipe captureしExecution Logへ。

Known Secretをwrite前redact。stdout textはIPC parserへ渡さない。

## 20. Terminal fencing / late message

RunnerService state updateはcurrent:

```text
runtime_instance_id
runner_instance_id
job_run_id
attempt_id
```

を条件にする。

Attempt terminal後またはRunner lost後に届くlate IPC/runtime response/resultはDB状態を変更しない。

Child terminal message後のobservation/runtime requestはreject/ignore-with-diagnosticし、success stateを上書きしない。

## 21. Temp / cleanup

```text
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

Payload commit後temp削除。Managed Artifactはput時durable copy済み。

Cleanup failureはJob conclusionを変更せずwarning/Event/maintenance cleanup候補。

Tempはsandboxではない。

## 22. Internal Job timeout

`timeout-minutes` internalのみ。未指定無期限。

1. deadline到達
2. `cancel_requested(reason=timeout)`
3. grace default10秒 configurable finite>=0
4. 未終了ならchild terminate
5. Attempt failure=`job_timeout`
6. Retry policy

## 23. Workflow Cancel vs Parent shutdown

Workflow Cancelは`cancel_requested(reason=workflow_cancel)`。

Parent正常shutdownはWorkflow cancelではない。New claim停止、bounded reap。Workflow `cancel_requested`は立てない。未完了Attemptは次起動時runner_lost Recovery。

## 24. Restart policy

- never: no automatic restart
- on_failure: unexpected exit restart
- window内max超過 -> restart_suppressed
- configured backoff
- normal parent shutdownはcount外

## 25. 非目標

CPU/RAM/GPU quota、本格sandbox、arbitrary shell、Pool global pause、Pool Action allow-list無し。

## 26. 受入条件

1. Integration Bootstrap roles/Windows spawn
2. Action Runner credential isolation
3. Pool config/runner_count/restart validation
4. heartbeat/main-loop/lost exactly once
5. atomic pool-matched claim
6. ready->start handshake
7. envelope malformed/unknown/duplicate terminal reject
8. exact start persistent input + Secret materialization
9. cancel reader while Action/runtime request waits
10. state get/set request-response
11. Artifact put/register/materialize request-response
12. log/progress validation
13. Step key mapping/open crash incomplete
14. ActionFailure/error terminal
15. result file exact path/size/digest/canonical JSON
16. result/error/exit combination matrix
17. stdout/stderr protocol separation/redaction
18. terminal/old Runner fencing
19. timeout/cancel
20. Parent restart runner_lost
21. temp cleanup/restart suppression
