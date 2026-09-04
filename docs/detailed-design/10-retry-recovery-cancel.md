# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03-runtime-and-scheduling.md`, `04-runner-and-ipc.md`, `06-reusable-workflows.md`, `07-external-and-human.md`, `08-persistence.md`

## 1. 目的

Structured Failure、Automatic/Manual Retry、backoff、Timeout、Runner/Runtime Recovery、Pause/Resume/Cancelを定義する。

## 2. 基本原則

1. Retryは新Attemptを作る。失敗Attemptを書き換えない。
2. Retryの永続Job Input/Definition/Action version bindingは固定。
3. Automatic RetryはYAMLで明示時のみ。
4. Manual Retryはfailed Job 1件を明示指定するMVP API。
5. Job timeout未指定は無期限。
6. Cancelはgraceful。公開force-success/force-kill APIなし。
7. Recoveryだけでcompleted Workflow Runを再openしない。
8. completed failed Runを再openできるのは認可されたManual Retryだけ。

## 3. Structured Failure

```text
category
code
message
retryable
details optional
occurred_at
```

category:

```text
execution
validation
protocol
timeout
runner
external
human
cancel
internal
```

## 4. 標準failureの既定retryable

| code | default |
| --- | --- |
| `action_exception` | true |
| `action_process_exit` | true |
| `job_timeout` | true |
| `runner_lost` | true |
| `external_lease_expired` (`fail` policy) | true |
| `child_workflow_start_failed` | true |
| `output_validation_failed` | false |
| `output_too_large` | false |
| `success_condition_failed` | false |
| `ipc_protocol_error` | false |
| `action_version_mismatch` | false |
| `human_rejected` | false |
| `cancelled` | false |
| `workflow_cycle_detected` | false |
| `internal_error` | false |

親定義/Actionがdomain failureを返す場合は明示`retryable`を設定できる。Engine標準codeのdefaultを親が無条件に上書きする仕組みはMVP必須にしない。

## 5. Retry YAML

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

未指定:

```text
automatic retry disabled
max-attempts = 1
```

`max-attempts`はautomatic schedulerが作成してよい**総Attempt番号の上限**。Manual Retryはこの上限を超えて新Attemptを開始できるが、`attempt_no >= max-attempts`になった後は追加Automatic Retryはしない。

## 6. Automatic Retry scheduling

Failed Attempt確定後:

1. Workflow/Job cancel無し
2. Retry policy存在
3. current `attempt_no < max-attempts`
4. Retry `if`を`failure` contextで評価
5. backoff算出
6. Jobを`queued`、`retry_not_before`設定
7. `retry_scheduled` Event

**この時点で次Attempt rowを作らない。**

実際のnext Attemptは:

- internal: Runner atomic claim
- external: External activation
- human: Human activation
- reusable: Child activation

で作る。

Retry condition expression errorは追加retryせずJob failureを維持しdiagnostic Eventを残す。

## 7. Backoff

```text
initial_seconds >=0
max_seconds >= initial_seconds
multiplier >=1
```

Attempt nのdelayは実装上overflowしない範囲でexponential計算し`max_seconds`でclip。JitterはMVPなし。

`retry_not_before`まではRunner/External/Human/Child activation対象外。Runnerを占有しない。

## 8. Manual Retry API

MVPは**Job指定のみ**。

```text
wf_retry(workflow_run_id, job_run_id, request_id?)
```

要求条件:

- Jobが`completed/failure`
- Jobが指定Workflow Runに属する
- Authorization pass
- Workflow Run conclusionは`failure`またはRunがまだnon-terminal
- cancel済みWorkflow Runはretry不可
- original Action version / Reusable bindingが現在実行可能であることを再確認

Job Input変更は不可。

## 9. completed failed Workflow Run の再open

Manual Retry対象Runが`completed/failure`なら同一transactionで:

1. `workflow_runs.run_attempt += 1`
2. Run `status = queued`、`conclusion = null`
3. `failure_json = null`, `output_json = null`, `completed_at = null`
4. target Jobのcurrent `conclusion/failure/completed_at`をclearし`queued`
5. `retry_not_before = now`
6. target Jobの**以前のAttempt rowsは維持**
7. target failureにより`blocked`になったdescendant Jobを再評価可能な`queued`状態へ戻す
8. Event + idempotency result

### 9.1 descendant reset範囲

Reset対象は:

- conclusion=`blocked`
- dependency graph上target Jobのdescendant
- blocked reasonがdependency failure系

Resetしない:

- `success`済みJob
- `failure`済み別Job
- 明示`if=false`で`skipped`済みJob
- `cancelled` Job

ResetされたJobはactivation時に全dependencyを再確認し、別failureが残れば再びblockedになれる。

Dynamic generated Jobもfull dependency graph上同規則。Dynamic expansion自体を再生成せず、既存generated Jobを使う。

## 10. Manual RetryとAttempt番号

Manual Retry request transactionではAttempt rowを作らない。次execution activationで`max(existing attempt_no)+1`を払う。

Auto retry上限到達後でもManual Retry可能。Manual attempt後は総attempt_noが既に`max-attempts`以上なら追加Auto Retryなし。

## 11. Retry Input / Secret / Artifact

固定:

- persisted Job Input
- Workflow Definition snapshot
- Workflow Input snapshot
- Dynamic item/iteration
- Reusable binding
- Action ID/version

Secretは同じreferenceを各internal Attempt起動直前に再materializeするためrotationを許容する（`02`）。

Failed Attempt Output/Artifactはcurrentとして使わない。新successful Attemptがcurrentになる。

## 12. Job Timeout

`timeout-minutes`明示時のみ。

期限:

1. Runner->Action Runner cancel(reason=timeout)
2. graceful grace（System default 10秒）
3. child未終了ならterminate
4. Attempt failure `job_timeout`
5. Automatic Retry判定

External lease durationとは別。

## 13. Runner lost

`04`のheartbeat/liveness threshold超過後、Recovery transactionでcurrent runner identityを再確認。

所有中running Attemptがまだ同じinstance/currentなら:

- Attempt -> completed/failure `runner_lost`
- Job failure stateへ
- Runner -> lost
- Retry policy評価

Completionとのraceはconditional updateでfirst terminal transition wins。

## 14. Runner restart

Defaultは`04`:

```text
on_failure
5 restarts / 300s
1s -> max30s exponential backoff
```

Crash loop -> `restart_suppressed`。Job retry policyとRunner restart policyは別物。

## 15. Parent Runtime restart

1. new `runtime_instance_id`
2. migration/Registry/Pool構築
3. non-terminal Run列挙
4. old runtime runnerをcurrent ownershipから除外
5. running internal Attemptをfencing/runner_lost recovery
6. External active leaseはexpires_atまで維持
7. Human pending review維持
8. waiting_child relation復元
9. queued/retry_not_before/concurrency/pause再評価
10. downstream activation漏れをidempotentに修復

Old Runtime/Runnerのlate updateは拒否。

## 16. Recovery idempotency

何回実行しても:

- terminal Attemptを再確定しない
- duplicate Attemptを作らない
- Dynamic expansionを重複しない
- Child Run/bindingを重複しない
- External Task/Human Reviewを同Attemptへ重複作成しない
- completed RunをRecovery理由で再openしない

DB unique/conditional updateを利用する。

## 17. Pause / Resume

Pause:

- Run `paused`
- running internal継続
- new internal/external/new Child activation禁止
- existing external submit/Human review/started Child進行は受理
- Lease expiry clock継続

Resume:

- pause flag解除
- activation再評価
- Attempt追加なし

同状態への再操作はidempotent no-op。

## 18. Cancel

Workflow cancel transaction/propagation:

- `cancel_requested=true`
- queued Job -> completed/cancelled
- waiting_external -> Lease invalidation + Attempt/Job cancelled
- waiting_review -> Review cancelled + Attempt/Job cancelled
- waiting_child -> Child cancel propagation
- running internal -> cancel request
- new activation/expansion禁止

Cancel後late external/human/runner completionはstate/ownershipに従い拒否またはcancelled結論へ収束する。

## 19. Force操作

Public Service/MCPに:

```text
force_success
force_complete
force_kill_runner
```

をMVPでは提供しない。

Timeout/parent shutdown等でRunnerがAction子Processをterminateする内部運用は別。

## 20. Workflow completion / Retry後

Retry後にtarget/descendantsが再terminalになったら通常`03`のWorkflow conclusionを再計算する。

過去`run_attempt`の履歴はAttempt/Eventから追跡可能。同一Workflow Run IDを維持する。

## 21. Events

```text
retry_scheduled
retry_started
retry_exhausted
manual_retry_requested
workflow_reopened_for_retry
runner_lost
runner_restart_suppressed
recovery_started/completed
cancel_requested/propagated
```

## 22. 受入条件

1. retryable default table
2. auto retry未指定無し
3. max attempts / condition / backoff
4. schedule時Attempt未作成
5. executor別next Attempt作成
6. Manual Retry Job-only
7. completed failed Run reopen + run_attempt
8. blocked descendant reset範囲
9. success/skipped/cancelled descendant非reset
10. retry Input/Reusable binding固定
11. Secret再materialize
12. timeout no-default + terminate
13. runner_lost race/fencing/retry
14. restart recovery exact
15. pause/resume idempotent
16. cancel全executor/child
17. Recovery terminal Run非reopen
