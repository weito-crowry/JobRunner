# 04. Runner / IPC 詳細設計

- Status: Draft v1.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `12`

## 1. 目的

Runner Process、Runner Pool、Integration Bootstrap、Heartbeat、Supervisor liveness、Common Action Runner、JSON Lines IPC v1、Secret materialization、Runtime Handle、large result、一時directory、restartを定義する。

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

Runner用HTTP/Broker必須無し。Action RunnerへDB path/provider/storage credentialを渡さない。

## 3. Integration Bootstrap

Parentはimport可能entrypointを1つ設定する。

```text
myapp.workflow_integration:configure_jobrunner
```

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

Registry semanticsは`01`。各Processで1 IDにつき1 current version。同じID二重登録reject。

Role:

- parent: Registry/Auth/Secrets/Store factory/public Service
- runner: Action metadata + Validator callable/Auth/Secrets/Store/RunnerService
- action_runner: Action callable解決だけ必須

Action RunnerへDB/Auth/Secrets/Store credentialを自動公開しない。

Snapshot version不一致=`action_version_mismatch|validator_version_mismatch`。

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
restart mode=on_failure
max_restarts=5
window=300s
backoff=1..30 x2
heartbeat=5s
lost=20s
main-loop-stale=15s
```

Validation:`lost>=heartbeat*2`, `stale>=heartbeat*2`。

Poolはrouting/Runner数/restart-livenessだけ。Action allow-list/resource quota/sandbox/Pool pause無し。

## 5. Runner identity/lifecycle

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

Parent lifecycle pipe EOFでnew claim停止。旧Runtime/instance updateはfencing reject。

## 6. Heartbeat / main-loop liveness

Main loopはbounded cycleごとmonotonic tick更新。Heartbeat Threadはtick fresh時だけheartbeat更新。Heavy Actionは別Processなのでheartbeatを止めない。

Supervisor:

- OS exit即時検知
- heartbeat interval以下でscan
- `now-last_heartbeat > lost_after` + current instance一致でlost候補
- Attempt ownership再確認後`runner_lost` exactly once

## 7. Atomic claim

Runnerは`runtime_instance_id/runner_id/runner_instance_id/pool`でclaim。

Candidate必須:

```text
executor=internal
status=queued
ready_at non-NULL
retry_not_before NULL or <= now
resolved runs-on == Runner pool
pending_input_json/bindings/digest present
same Workflow Run running internal無し
```

`03/08` 1 transactionで:

- Job running
- new Attempt
- pending persistent Input/bindings/digest exact copy
- Runner ownership/current Job
- Execution Log row/file metadata
- Event

を確定。

1 Runner同時1 Job。

## 8. Action callable

```python
def action(input_data) -> AnyJson: ...
def action(input_data, runtime) -> AnyJson: ...
```

sync/async対応。

```python
raise ActionFailure(
    code="rate_limited",
    message="upstream temporarily unavailable",
    retryable=True,
    details={"provider": "example"},
)
```

Unhandled exception=`action_exception`, retryable=false。

## 9. IPC transport v1

Dedicated bidirectional local byte transport。stdout/stderr分離。

```text
UTF-8
one JSON object per line
LF
protocol="jobrunner.action-ipc.v1"
```

Envelope:

```json
{
  "protocol": "jobrunner.action-ipc.v1",
  "type": "...",
  "request_id": null,
  "payload": {}
}
```

- protocol/type/payload required
- payload object
- request_idはRuntime request/responseだけ必須
- unknown top-level/type reject
- malformed UTF-8/JSON=`ipc_protocol_error`
- Output本体はIPCへ送らない

## 10. Handshake

1. spawn Child
2. Child `role=action_runner` bootstrap
3. exactly one `ready`
4. Runner current ownership確認
5. exactly one `start`
6. Action execute
7. zero+ observation/runtime requests
8. exactly one terminal `result` or `error`
9. optional `exiting`
10. OS exit

ready前message/start重複/terminal重複=`ipc_protocol_error`。

`ready`:

```text
pid integer
protocol_version=1
```

## 11. `start` / Secret materialization

Payload:

```text
action_id
action_version
workflow_run_id
job_run_id
attempt_id
persistent_input: object
secret_bindings: array<{pointer,name}>
secrets: object<string,string>
work_dir: absolute path
runtime.cancel_requested: boolean
```

Rules:

- persistent_input/bindings/digestのSource of TruthはAttempt snapshot
- bindingsは`02` RFC6901 canonical form
- `secrets` key集合はbindingsで参照されるSecret name集合とexact一致
- Secret valueは`12` non-empty string
- binding pointer先persistent valueはcanonical `${{ secrets.NAME }}` string
- unbound marker-like literalは通常stringとして扱う
- binding不整合=`ipc_protocol_error`/`secret_binding_invalid`

ChildはAction呼出直前にpersistent_inputをmemory copyし、binding pointerだけを`secrets[name]`へ置換してexecution inputを作る。

Execution input/Secret値をfile/log/debug dumpへ保存しない。

RunnerはSecret missing/invalidをChild spawn前に検出し、`secret_not_found|secret_value_invalid`でAttempt failureにできる。

## 12. `cancel_requested`

Payload:

```text
reason=workflow_cancel|timeout
requested_at RFC3339 UTC
```

Child reader loopはAction中も受信しRuntime Handle cancel flag更新。

- workflow_cancel -> Job cancelled
- timeout -> grace後terminateなら`job_timeout`

Parent normal shutdownはcancel messageではない。

## 13. Runtime Handle request/response

Child requestはunique non-empty `request_id`。Runnerはexactly one response。

Success:

```json
{"type":"runtime_response","request_id":"req","payload":{"ok":true,"result":{}}}
```

Failure:

```json
{"type":"runtime_response","request_id":"req","payload":{"ok":false,"error":{"code":"...","message":"...","retryable":false,"details":null}}}
```

Outstanding request中もcancel受信。Terminal後new request禁止。

### 13.1 `state_get`

Request=`name` non-empty。

Response=`found/value/revision`。成功利用でAttempt `reuse_eligible=false`。

### 13.2 `state_set`

Request=`name + JSON-compatible value`。Response=`revision>=1`。RunnerServiceがSecretGuard + current/history transaction。

### 13.3 `artifact_put_file`

Request:

```text
name
relative_work_path
media_type optional
metadata object optional
```

Response=canonical ArtifactRef。

### 13.4 `artifact_register_external`

Request=`name,uri,media_type?,size_bytes?,digest?,metadata?`。Response=External ArtifactRef。Core fetch無し。

### 13.5 `artifact_materialize`

Request=`artifact_ref`。Response=`relative_work_path`。DB re-resolve + `09/12` authorization。External reject。Persistent Input外materialize -> reuse ineligible。

## 14. Observation messages

### `log`

```text
level=debug|info|warning|error
message string
```

### `progress`

```text
current finite>=0
total finite>0 optional
message optional
unit optional
```

Totalありならcurrent<=total。`09`のJob progress modeに従い反映する。

### `step_started`

```text
step_key non-empty unique/name/metadata?
```

Runnerがsequence/DB Step ID採番。

### `step_finished`

```text
step_key existing open
conclusion=success|failure|cancelled
metadata optional
```

Unknown/closed key=protocol error。Open Stepは異常終了時incomplete。

## 15. Terminal `error`

Payload:

```text
category
code
message
retryable
details optional
```

ActionFailure -> category=action。Unhandled -> action_exception/false。Version mismatchはvalidation failure。

RunnerはSecretGuardしてAttempt failure。Secret混入errorはsafe replacementへ変換。

## 16. Terminal `result` / result file

Child:

1. return JSON-compatible確認
2. `work_dir/.jobrunner/result.json.tmp`
3. canonical-json-v1 write
4. close/fsync相当 + atomic rename
5. size/SHA-256
6. result message

Payload:

```text
relative_path=".jobrunner/result.json"
size_bytes>=0
sha256 lowercase hex
```

Runner:

1. exact reserved path/symlink escape
2. existence/size/digest
3. JSON deserialize
4. Output Schema
5. Validator with **persistent_input**
6. success_if
7. SecretGuard
8. PayloadStore

Validatorへexecution input/Secret valueを渡さない。

## 17. `exiting` / exit matrix

Optional `exiting.reason=result|error`。Source of Truthではない。

- result + exit0 + validation success -> success候補
- result + nonzero -> action_process_exit, result非採用
- error + any exit -> error failure採用、exit0はdiagnostic
- no terminal + nonzero -> action_process_exit
- no terminal + exit0 -> action_result_missing
- result+error -> ipc_protocol_error

## 18. stdout/stderr / Execution Log

Child stdout/stderrは別pipeでAttempt Execution Logへ。Known Secretはwrite前redact。IPC parserへ渡さない。

Log verbosity/全executor共通規則は`09`。

## 19. Terminal fencing

State updateはcurrent runtime/runner/job/attempt ownershipを条件にする。

Attempt terminal/Runner lost後のlate IPC/result/runtime responseはDB状態を変更しない。

## 20. Temp lifecycle

```text
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

Payload commit後削除。Managed Artifactはput時durable copy済み。Cleanup failureはconclusion不変、maintenance cleanup候補。Sandboxではない。

## 21. Internal timeout

`timeout-minutes` internalのみ。未指定無期限。

1. deadline
2. cancel_requested(timeout)
3. grace default10秒 configurable finite>=0
4. terminate
5. `job_timeout`
6. Retry policy

## 22. Parent shutdown / restart

Workflow CancelとParent shutdownを分離。

Parent正常shutdown=New claim停止/bounded reap/cancel_requested無し。未完了Attemptは次起動runner_lost Recovery。

Restart policy=`never|on_failure`, default on_failure。Window max5/300s、backoff1..30x2 default。Crash loop -> restart_suppressed。

## 23. 非目標

CPU/RAM/GPU quota、本格sandbox、arbitrary shell、Pool global pause、Pool Action allow-list無し。

## 24. 受入条件

1. Bootstrap roles/Registry one-current semantics
2. Pool config/liveness/restart
3. claim requires ready_at + pending snapshot
4. pending snapshot exact Attempt copy
5. ready->start handshake/envelope errors
6. start secret_bindings exact materialization
7. marker-like unbound literal not materialized
8. Secret missing/invalid pre-child failure
9. cancel while Action/request waits
10. Runtime Handle correlation
11. Artifact/state operations
12. progress/Step validation
13. ActionFailure/error
14. result file canonical path/size/digest
15. Validator gets persistent input only
16. result/error/exit matrix
17. stdout/redaction/common Log
18. fencing/timeout/temp
19. Parent restart runner_lost/restart suppression
