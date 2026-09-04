# 04. Runner / IPC 詳細設計

- Status: Draft v1.9
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `12`

## 1. 目的

Runner Process、Runner Pool、Integration Bootstrap、Action invocation contract、Heartbeat、Supervisor liveness、Common Action Runner、JSON Lines IPC v1、Secret materialization、Runtime Handle、Step/telemetry、large result、一時directory、timeout/cancel fencing、restartを定義する。

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

## 3. Integration Bootstrap / Registry callable contract

Parentはimport可能entrypointを1つ設定する。

```text
myapp.workflow_integration:configure_jobrunner
```

```python
def configure_jobrunner(builder, role: str) -> None: ...
```

Role=`parent|runner|action_runner`。

Canonical Action registration:

```python
builder.register_action(
    action_id,
    version,
    callable,
    uses_runtime=False,
    metadata=None,
)
```

Canonical Validator registration:

```python
builder.register_validator(
    validator_id,
    version,
    callable,
)
```

Other builder operations:

```text
set_authorization_provider_factory(factory)
set_secrets_provider_factory(factory)
set_payload_store_factory(factory) optional
set_artifact_store_factory(factory) optional
```

Registry semanticsは`01`。各Processで1 IDにつき1 current version。同じID二重登録reject。

### 3.1 Action invocation

`uses_runtime` はboolean、default false。Coreがcallable signatureを見てRuntime Handle要否を推測しない。

```text
uses_runtime=false -> callable(execution_input)
uses_runtime=true  -> callable(execution_input, runtime_handle)
```

- positional callで実行
- callableは指定argument countで呼出可能でなければregistration/bootstrap error
- varargsでも`uses_runtime`がSource of Truth
- returnはJSON-compatible valueまたはawaitable producing JSON-compatible value
- awaitableはAction Runner専用event loopで完了までawait
- sync callableがawaitableを返す形も許可
- Action child内でparent event loopを継承しない

### 3.2 Validator invocation

ValidatorはMVPでは同期・軽量callableだけ。

```python
validator(result_value, persistent_input) -> ValidationResult
```

Awaitable return=`validator_contract_error`。重い/非同期validationは通常Job Actionへ分離。

Canonical `ValidationResult`:

```text
valid: boolean
code: non-empty string optional
message: string default ""
retryable: boolean default false
details: JSON-compatible optional
```

`valid=false`かつcode省略時はCore code=`domain_validation_failed`。`valid=true`側の補助fieldはresult判定へ影響しない。

### 3.3 Role

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
restart_policy.backoff.initial_seconds finite>=0
restart_policy.backoff.max_seconds finite>=initial
restart_policy.backoff.multiplier finite>=1
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

- `runner_id`=Pool内の論理slot。Parent Runtime存続中は同じslotをrestart後も再利用
- `runner_instance_id`=各OS Process instanceごとに新規発行
- `runtime_instance_id`=Parent Runtime起動ごとに新規発行

Status=`starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed`。

Parent lifecycle pipe EOFでnew claim停止。旧Runtime/instance updateはfencing reject。

## 6. Heartbeat / main-loop liveness

Main loopはbounded cycleごとmonotonic tick更新。Heartbeat Threadはtick fresh時だけheartbeat更新。Heavy Actionは別Processなのでheartbeatを止めない。

Supervisor scanは`heartbeat_interval_seconds`以下の間隔で行う。

Canonical liveness:

```text
last_heartbeat_at non-NULL:
  now - last_heartbeat_at > runner_lost_after_seconds -> lost candidate

last_heartbeat_at NULL:
  now - started_at > runner_lost_after_seconds -> startup_lost candidate
```

つまりRunnerが起動途中で固まり**一度もheartbeatを出さない場合もlost判定できる**。`started_at`はOS Process spawn成功後、instance row作成時のcanonical timestamp。

`main_loop_tick_at`はheartbeat Threadが更新可否を判断する内部freshness情報。Main loop tickが`main_loop_stale_after_seconds`より古い場合、Heartbeat Threadは新しいheartbeatを書かず、最終的に上記lost判定へ到達させる。

Supervisor:

1. OS process exitはscan待ちせず可能な限り即時検知
2. liveness threshold到達時はcurrent `runtime_instance_id/runner_instance_id`を再確認
3. current Attempt ownershipを再確認
4. まだalive processが残るlost判定ではterminate/reapを試行
5. Attempt ownershipがあれば`runner_lost`をexactly once terminalizeし`10` Retryへ
6. Runner instanceを`lost`または`stopped`へ確定
7. §22 restart policyを適用

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

`03/08` 1 transactionでJob running + new Attempt + pending snapshot copy + Runner ownership + Execution Log + Event。

1 Runner同時1 Job。

`timeout-minutes`がある場合、**claim transaction成功直後にRunner monotonic clockでAttempt timeout originを固定**する。Action Runner spawn前に開始し、terminal Attempt commitまで同じdeadlineを使う。DB wall-clock timestampとしてtimeout deadlineを永続化する必要はない。Runner lost後は同Attemptを継続せず`10` Recoveryへ移るためである。

## 8. Action failure contract

```python
raise ActionFailure(
    code="rate_limited",
    message="upstream temporarily unavailable",
    retryable=True,
    details={"provider": "example"},
)
```

Contract:

- code=non-empty string
- message=string
- retryable=boolean
- details=nullまたはJSON-compatible

Contract違反=`action_contract_error`, retryable=false。Unhandled exception=`action_exception`, retryable=false。

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

`ready`: `pid integer>0`, `protocol_version=1`。

`timeout-minutes`はこのhandshake/bootstrap時間も含む。Deadlineが`ready`前に到達した場合はcooperative `cancel_requested`を送れるIPC状態ではないため、RunnerはChildをterminateして`job_timeout`へ収束してよい。これはpublic force-cancel APIではなく、既に成立したJob timeoutのcleanupである。

Workflow cancelが`ready`前にcommitした場合もnew Action開始を行わずChildを終了しAttemptをcancelledへ収束する。

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

Runnerは`12`どおりunique Secret nameをExactly once/nameで解決してAttempt Secret Setを作る。

Rules:

- persistent_input/bindings/digest Source of Truth=Attempt snapshot
- bindings=`02` RFC6901 canonical form
- `secrets` key集合=bindings Secret name集合exact一致
- same Secret nameの複数bindingは同じAttempt Secret Set valueを使う
- Secret value=`12` non-empty string
- binding pointer先persistent value=canonical `${{ secrets.NAME }}` string
- unbound marker-like literal=通常string
- binding不整合=`ipc_protocol_error|secret_binding_invalid`

ChildはAction呼出直前にpersistent_inputをmemory copyしbinding pointerだけをSecret valueへ置換してexecution inputを作る。

Execution input/Secret値をfile/log/debug dumpへ保存しない。

Non-empty Secret bindingを持つAttemptは最初からResult Reuse不適格として扱う。

## 12. `cancel_requested`

Payload:

```text
reason=workflow_cancel|timeout
requested_at canonical UTC timestamp
```

Child reader loopはAction中も受信しRuntime Handle cancel flag更新。

- workflow_cancel -> graceful cancellation request
- timeout -> timeout deadline already成立済み、graceful cleanup request

Parent normal shutdownはcancel messageではない。

## 13. Runtime Handle request/response

Child requestはunique non-empty `request_id`。Runnerはexactly one response。Outstanding request中もcancel受信。Terminal後new request禁止。

### 13.1 `state_get`

Request=`name` non-empty。Response=`found/value/revision`。Callが正常処理された時点でfound true/falseに関係なくAttempt `reuse_eligible=false`。

### 13.2 `state_set`

Request=`name + JSON-compatible value`。Response=`revision>=1`。

RunnerServiceがAuthorization + SecretGuard + current/history transactionをrequest時点で即時commit。

- Attempt successまで保留しない
- later failure/cancel/timeout/runner_lostでもrollbackしない
- historyへproducer Job/Attemptを記録
- **その時点でopen StepがあればそのStep IDもproducer `step_id`として記録し、open Stepが無ければNULL**
- 成功利用でAttempt `reuse_eligible=false`

業務transactionが必要なら専用Action/親DB transaction等で設計する。

### 13.3 `artifact_put_file`

Request=`name,relative_work_path,media_type?,metadata?`。Response=canonical ArtifactRef。

### 13.4 `artifact_register_external`

Request=`name,uri,media_type?,size_bytes?,digest?,metadata?`。Response=External ArtifactRef。Core fetch無し。

### 13.5 `artifact_materialize`

Request=`artifact_ref`。Response=`relative_work_path`。DB re-resolve + `09/12` authorization。External reject。Persistent Input外materialize -> reuse ineligible。

## 14. Observation messages / Step model

Persistent telemetry strings/metadataは`12`のredactionを通してから保存する。Known Secretを含む表示fieldは`[REDACTED]`化し、telemetryだけを理由にAttempt failureにはしない。

**MVPでは1 Attemptにつき同時にopen可能なStepは最大1つ。** Stepはネスト/並列化しない。これによりState history等の「現在のStep」対応を一意にする。

### 14.1 `log`

```text
level=debug|info|warning|error
message string
```

### 14.2 `progress`

```text
current finite>=0
total finite>0 optional
message string optional
unit string optional
```

Totalありならcurrent<=total。`09` progress mode。

`message/unit`はredact後保存。

### 14.3 `step_started`

Payload:

```text
step_key non-empty string
name non-empty string optional
metadata JSON object optional
```

Rules:

- `step_key`はAttempt内unique correlation key
- 既にopen Stepが存在する状態でnew `step_started` -> `ipc_protocol_error`
- 同じkeyの2回目startも`ipc_protocol_error`
- DB `job_steps.name = provided name or step_key`
- display nameはredact後保存
- `step_key`自体はcorrelation用で、DBに専用columnは持たない
- metadataはJSON objectのみ。redact後に`start_metadata_json`へ保存

Runnerがsequence/DB Step ID採番し、current open Stepとしてmemory保持する。

### 14.4 `step_finished`

Payload:

```text
step_key = current open Step key
conclusion=success|failure|cancelled
metadata JSON object optional
```

Open Step無し/別key/既にclosed key=`ipc_protocol_error`。Finish metadataはredact後に`finish_metadata_json`へ保存する。Finish時にstart metadataを上書きしない。

Start/finish metadata未指定なら対応DB column=`NULL`。Finish commit後current open Stepをclearする。

Open StepがAction process異常終了/timeout/cancelで正常finishされなかった場合、Runnerが`conclusion=incomplete`へ閉じ、`finish_metadata_json=NULL`を保持してopen Stepをclearする。

## 15. Terminal `error`

Payload=`category/code/message/retryable/details?`。

ActionFailure -> category=action。Unhandled -> action_exception/false。Version mismatchはvalidation failure。

RunnerはSecretGuard。Secret混入errorはsafe replacement。

## 16. Terminal `result` / result file

Child:

1. return JSON-compatible確認
2. `work_dir/.jobrunner/result.json.tmp`
3. canonical-json-v1 write
4. close/fsync相当 + atomic rename
5. size/SHA-256
6. result message

Payload=`relative_path=".jobrunner/result.json"`, `size_bytes`, `sha256 lowercase hex64`。

Runner result validation前にcurrent ownership + timeout/cancel fenceを再確認する。

1. timeout deadline成立済み -> result破棄、`10` job_timeoutへ
2. Workflow cancel commit済み -> result破棄、cancelledへ
3. 上記無し -> reserved path/symlink escape
4. existence/size/digest
5. JSON deserialize
6. Output Schema
7. Validator with persistent_input
8. success_if
9. SecretGuard
10. PayloadStore

`timeout-minutes`のdeadlineはresult受信時で止めない。Validation/PayloadStore処理中も継続し、**Attempt terminal commit直前にmonotonic deadlineを再確認**する。Deadline成立済みならsuccessをcommitせず`job_timeout`へ収束する。

Result commit transactionでもcancel/current ownershipを再確認し、validation中にCancelがcommitしたraceでsuccessを確定しない。

## 17. `exiting` / exit matrix

Optional `exiting.reason=result|error`。Source of Truthではない。

- result + exit0 + validation success + fence clear + timeout deadline前terminal commit -> success候補
- result + nonzero -> action_process_exit, result非採用
- error + any exit -> error failure採用、exit0 diagnostic
- no terminal + nonzero -> action_process_exit
- no terminal + exit0 -> action_result_missing
- result+error -> ipc_protocol_error
- timeout flag成立後は上記より`job_timeout`が優先
- Workflow cancel commit済みならsuccess resultよりcancelledが優先

## 18. stdout/stderr / Execution Log

Child stdout/stderrは別pipeの**raw bytes**でRunnerへ渡す。

Runnerは`12`のAttempt Secret Set byte streaming redactorを適用する。

- Secretがpipe chunk境界をまたいでも検出
- redactionはUTF-8 decode前のbytes段階
- redaction後bytesだけをdecoder/Execution Log writerへ渡す
- EOFでmatcher suffix flush
- raw pre-redaction sink/file無し

IPC parserへstdout/stderrを渡さない。Log policy=`09`。

## 19. Terminal fencing

State updateはcurrent runtime/runner/job/attempt ownership条件。

Attempt terminal/Runner lost後late messageはDB変更不可。

Workflow cancel/result raceとtimeout/result raceは`10`のcommit/deadline境界を使う。

## 20. Temp lifecycle

`runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/`。

Payload commit後削除。Managed Artifactはput時durable copy済み。Cleanup failureはconclusion不変。Sandboxではない。

## 21. Internal timeout

`timeout-minutes` internalのみ、未指定無期限。

Canonical timeout interval:

```text
start = internal Attempt claim transaction成功直後のRunner monotonic time
end   = Attempt terminal DB commit
```

したがって以下を全て含む:

- Action Runner process spawn
- `role=action_runner` bootstrap
- ready/start handshake
- Secret materialization/start準備
- Action本体
- result/error transport
- result file検証
- Schema/Validator/success_if/SecretGuard
- PayloadStore prepare/final terminalization

Deadline到達時点でtimeout outcomeが成立する。

1. deadline到達
2. timeout flag
3. Childがready/start済みなら`cancel(timeout)`でcooperative cleanup要求
4. ready前/IPC不可ならChild terminate可
5. grace default10秒はcleanup猶予
6. grace中result/errorはcleanup signalとして受けられるがsuccess resultは採用しない
7. 未終了ならterminate
8. open Stepはincompleteへclose
9. Attempt=`job_timeout`
10. Retry policy

Graceは成功猶予ではない。**Deadline前にAttempt terminal success commit済みならtimeout処理しない。** Resultをdeadline前に受け取っただけでは不十分で、terminal commitがdeadline後ならtimeoutを優先する。

## 22. Parent shutdown / Runner restart policy

### 22.1 Parent正常shutdown

Parent正常shutdownでは:

1. Scheduling/new claim停止
2. Runnerへ`stopping`通知/lifecycle pipe close
3. bounded reap
4. `cancel_requested`をWorkflowへ自動設定しない
5. 正常に終了したRunnerは`stopped`
6. shutdown開始後の意図したRunner終了にはautomatic restartを掛けない

未完了internal Attemptは次Parent起動時に`runner_lost` Recoveryへ収束する。Parent restartでは新しい`runtime_instance_id`となり、前Runtimeのrestart budgetを継承しない。

### 22.2 Failure classification

Runner logical slotについて以下を**failure**としてrestart policy対象にする:

- OS process unexpected exit
- non-zero exit
- `starting/idle/claiming/running`中のliveness lost
- first heartbeat前のstartup lost
- Supervisorがmain-loop stale経由でlost確定した場合

以下はfailure restart対象外:

- Parent正常shutdownに伴う停止
- Supervisorが明示的に`stopping`へ移したplanned stop
- Parent Runtime自体の終了後に観測された旧instance

Exit code 0でもplanned stopでなければunexpected exitとしてfailure扱いする。

### 22.3 Restart mode

`restart_policy.mode=never`:

- failure確定後automatic restartしない
- logical slotを`restart_suppressed`として可視化
- `runner_restarts`へ`suppressed=1, reason=policy_never`記録

`restart_policy.mode=on_failure`:

- §22.2 failure時のみautomatic restart候補
- planned stop/normal shutdownではrestartしない

### 22.4 Rolling restart window

Budgetは同じ:

```text
runtime_instance_id + pool_name + runner_id
```

のlogical slot単位。

Failure時刻を`now`とし、`runner_restarts.created_at > now - window_seconds` の**過去のautomatic restart launch/suppression判定記録**をwindow内履歴として扱う。

`max_restarts`はそのwindow内で許すautomatic restart **launch回数**。

- `max_restarts=0` -> 最初のfailureからrestartせずsuppressed
- window内の既存`suppressed=0` restart launch数 `< max_restarts` -> 次restart可
- 既存launch数 `>= max_restarts` -> restartせず`suppressed=1, reason=restart_limit_exceeded`
- 古いrestart recordがrolling window外へ出れば自然にbudgetが回復する
- stable uptimeによる別のmanual reset counterは持たない

Parent Runtime再起動はruntime_instance_idが変わるため新budgetとなる。

### 22.5 Restart ordinal / backoff

Restartを許可する場合、window内の既存successful launch record数を`k`として:

```text
restart_ordinal = k + 1
raw_delay = initial_seconds * multiplier ** (restart_ordinal - 1)
delay = min(max_seconds, raw_delay)
scheduled_for = now + delay
```

Retry backoffと同様、overflow-safe saturating calculationを使いNaN/Infinity/OverflowErrorを出さない。

`runner_restarts` rowを先に作り、`scheduled_for`を保存する。Delay経過前にParent shutdownした場合は新Runnerを起動せず、そのrecordはhistorical scheduleとして残してよい。新Parent Runtimeでは再利用しない。

Scheduled time到達後:

1. Parent Runtimeがまだcurrent
2. logical slotがrestart_suppressedでない
3. Pool config上slotが必要

を確認してnew `runner_instance_id`をspawnする。Spawn開始成功時`started_runner_instance_id`を記録する。

Spawn自体が即失敗/新instanceがstartup_lostした場合も新しいfailureとして同じrolling windowへ入り、次のbudget/backoff判定を行う。

### 22.6 Restart suppressed visibility

Suppressed時:

- logical slotのcurrent visible state=`restart_suppressed`
- reason=`policy_never|restart_limit_exceeded`
- Event/Execution diagnosticを残す
- Runtimeはそのslotを自動復活させない
- 他slot/Poolは通常継続する

MVPにpublic「restart budget reset」APIは持たない。復旧はParent configuration修正後のParent Runtime再起動、または親内部運用で行う。

## 23. 非目標

CPU/RAM/GPU quota、本格sandbox、arbitrary shell、Pool global pause、Pool Action allow-list無し。

## 24. 受入条件

1. Bootstrap roles/Registry one-current semantics
2. register_action uses_runtime exact invocation
3. sync/async/returned-awaitable Action
4. invalid Action arity registration reject
5. Validator sync-only/awaitable reject/ValidationResult defaults
6. ActionFailure contract validation
7. Pool config strict validation
8. startup before first heartbeat lost detection from started_at
9. heartbeat/main-loop stale liveness
10. planned shutdown vs unexpected exit classification
11. restart mode never/on_failure exact behavior
12. max_restarts=0 and rolling window budget
13. stable elapsed window naturally restores restart budget
14. restart ordinal/backoff saturating calculation
15. scheduled restart cancelled by Parent shutdown/no cross-runtime reuse
16. restart_suppressed visibility/reason
17. claim ready_at + pending snapshot
18. timeout origin starts immediately after claim before child spawn
19. timeout covers bootstrap/handshake/action/result validation until terminal commit
20. pre-ready timeout/cancel cleanup
21. unique Secret resolution exactly once/name/Attempt
22. same Secret multiple bindings exact same value
23. non-empty Secret binding reuse ineligible
24. ready->start/IPC errors
25. cancel while Action/request waits
26. Runtime Handle correlation
27. state_get found/missing both reuse ineligible
28. state_set immediate nonrollback + current Step producer association
29. Artifact operations
30. progress telemetry redaction
31. at most one open Step / nested start reject
32. Step key/name/start-finish metadata exact columns
33. open Step incomplete close
34. result file/exit matrix
35. deadline-before-terminal-commit result discard
36. cancel commit result discard
37. stdout/stderr streaming byte redaction across chunks
38. no raw pre-redaction sink
39. fencing/temp/restart
