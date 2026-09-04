# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `04`, `05`, `07`, `08`, `11`

## 1. 基本原則

1. Retry=new Attempt。
2. Failed Attempt rowを書き換えない。
3. Retry Input/Secret bindings/Definition/Action/Validator/Dynamic iteration/Reusable binding固定。
4. Automatic Retryは`retry` block明示時のみ。
5. Manual Retryはfailed concrete Job指定。
6. Retryにはfailed Attempt persistent Input snapshot必須。
7. Internal timeoutのみ。未指定無期限。
8. Cancel=graceful。Cancel commit後の成功結果は採用しない。
9. Recoveryだけでcompleted Run reopen無し。
10. Manual Retry後successful descendantは`03/05` strict reuse必須。
11. completed RunをManual Retryでreopenする場合はConcurrency slotを再取得する。

## 2. Structured Failure

```text
category
code
message
retryable
details optional
occurred_at
```

`retryable`必須bool。Missingを暗黙trueにしない。

## 3. Core retryability defaults

| code | default retryable |
| --- | --- |
| runner_lost | true |
| job_timeout | true |
| external_lease_expired(fail) | true |
| payload_storage_failed | true |
| action_process_exit | false |
| action_exception | false |
| ipc_protocol_error | false |
| result_file_invalid | false |
| output_validation_failed | false |
| validator_exception | false |
| domain_validation_failed | Validator value |
| success_condition_failed | false |
| action_version_mismatch | false |
| validator_version_mismatch | false |
| human_rejected | false |
| secret_not_found/secret_value_invalid | false |
| secret_value_persistence_blocked | false |
| successful_job_result_not_reusable | false |
| dynamic_expansion_not_reusable | false |
| retry_input_unavailable | false |
| cancelled/internal_error | false |

Parent `ActionFailure`/Validator supplied retryableはそのまま使用。

`retryable=false`でもYAML `retry.if`が明示trueならAttempt failureはAutomatic Retry可能。ただしAttempt自体が無い`retry_input_unavailable`系はRetry不能。

## 4. Retry YAML

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

Absent:

```text
automatic retry disabled
max attempts=1
```

Present defaults:

```text
max-attempts=2
if=${{ failure.retryable }}
initial=0
max=initial
multiplier=1
```

Validation:

- max-attempts integer 2..signed64 max
- if CEL boolean
- initial finite>=0
- max finite>=initial
- multiplier finite>=1
- unknown key reject

Delay for next Attempt `n>=2` の数学的定義:

```text
raw = initial_seconds * multiplier ** (n-2)
delay = min(max_seconds, raw)
```

ただし実装は**overflow-safe saturating calculation**を必須とする。

- `initial_seconds == 0` -> delay=0
- `multiplier == 1` -> delay=min(max, initial)
- それ以外も、rawをIEEE floatで先に無制限計算して`inf/OverflowError`を出してからclampしてはならない
- `max_seconds`へ到達すると数学的に判定できた時点で即`max_seconds`を返してよい
- attempt番号がsigned64上限近くでもdelayはfiniteかつ`0 <= delay <= max_seconds`
- NaN/Infinityを生成/保存しない
- O(n)回の乗算を要求しない。大きい指数ではlog/上限比較等のsaturating判定を使ってよい

Jitter無し。

## 5. Retry pending snapshot

Retry request/schedule時に基準failed Attemptから:

```text
input_json
secret_bindings_json
input_digest
```

をJob `pending_*`へexact copyする。

同じtransactionでJobをnon-terminal queued stateへ戻す:

```text
status = queued
conclusion = NULL
completed_at = NULL
ready_at = retry schedule/request time
retry_not_before = due time or NULL
```

`current_attempt_id` はnew Attempt作成までは基準となったlatest failed Attemptを指してよい。前Attemptのfailure詳細はAttempt履歴と`current_failure_json`へ残してよく、queued Jobの`conclusion`とは別物として扱う。

**new Attemptはまだ作らない。**

Due判定は `now >= retry_not_before`。`retry_not_before=NULL`は即時due。

Due後:

- internal -> Runner claimがpending snapshotをnew Attemptへcopy
- external -> scheduler activationがnew Attempt + Taskへcopy
- human -> new Attempt + Reviewへcopy
- reusable -> new Attempt + Child Runへcopy

Attempt作成後pending fieldsはclear可能。

## 6. Automatic Retry flow

Failed Attempt terminal時:

1. Workflow cancel確認
2. current attempt_no < max-attempts
3. Retry `if`を`failure` contextで評価
4. false -> Job completed/failure
5. true -> overflow-safe delay計算
6. §5 pending snapshot + non-terminal Job state作成
7. `retry_not_before`保存
8. Maintenance notify
9. `retry_scheduled` Event/Execution Log

Retry condition errorはRetryせず元failure維持 + diagnostic/Event。

External submit validation/domain failureもこのflow。Submit transaction内でnew Attempt/Taskは作らない。

## 7. Manual Retry eligibility

`wf_retry(workflow_run_id, job_run_id)`。

Required:

- target Job conclusion=failure
- terminal failed Attempt>=1
- latest failed Attempt input/bindings/digest存在
- Workflow Run conclusion=failure またはnon-terminal
- snapshot Action/Validator/Reusable binding version available

Reject:

- success/cancelled Run
- Dynamic expansion failureでAttempt無し
- activation/Input resolution等Pre-Attempt failure

Attempt/Input無し=`retry_input_unavailable`, retryable=false。

## 8. Manual Retry on non-terminal Run

Runが既にnon-terminalならConcurrency holder状態は変更しない。

1 transactionで:

1. eligibility/idempotency
2. target Jobへ§5 pending snapshot + non-terminal state、`retry_not_before=NULL`
3. blocked/skipped descendantsをactivation再評価対象へ戻す
4. successful descendants `reuse_check_pending=1`
5. successful Dynamic group/expanded rowsを`05` reuse check対象
6. Event/Execution Log/idempotency result

Run `run_attempt` はincrementしない。これは同じRun attemptの途中で行うJob再試行である。

## 9. Completed failure Run reopen

Completed `conclusion=failure` RunをManual Retryする場合は**同じWorkflow Runを新しいrun_attemptとしてreopen**する。

Concurrencyを持たないRunはそのままreopen可能。

Concurrencyを持つRunは`08`のWorkflow-scoped concurrency規則でslotを再取得する。同一transaction内でactive holderを再計算する。

- capacityあり -> `status=running`, `wait_reason=NULL`, `concurrency_queued_at=NULL`
- capacity無し + `on-limit=queue` -> `status=queued`, `wait_reason=concurrency`, `concurrency_queued_at=retry transaction time`
- capacity無し + `on-limit=reject` -> `concurrency_limit_reached` conflict。**何も変更せず**retry request失敗

Reopenをcommitする場合、同じtransactionで:

1. `run_attempt += 1`
2. `conclusion=NULL`
3. `completed_at=NULL`
4. `failure_json=NULL`
5. Workflow Output storage fieldsを全てNULLにして旧Workflow Outputをcurrent公開対象から外す
6. `cancel_requested=0`, `pause_requested=0`
7. target Jobへ§5 pending snapshot + non-terminal state、`retry_not_before=NULL`
8. blocked/skipped descendants activation再評価対象
9. whole-skipped Dynamic expansionは`05`規則でreset可能
10. successful descendants `reuse_check_pending=1`
11. successful Dynamic expansionをreuse check対象
12. Event/Execution Log/idempotency result

`started_at`は最初のRun開始時刻として保持する。Past Attempt/Job Output/Artifact/Event/Logは削除しない。

Concurrency queue中でもtarget pending snapshotは保持し、slot取得までnew Attemptを作らない。

Manual RetryでConcurrency queueへ入る場合、元のRun `created_at`をwaiting orderへ使わず、新しい`concurrency_queued_at`を使う。

Manual Retryはautomatic `max-attempts` budgetをリセットしない。New Attempt番号=existing max+1。

## 10. Target Retry固定値

Target Retryで再評価しない:

- `with`
- upstream Output/state
- Secret binding name/path
- Dynamic item/iteration/key/order
- continue-on-error
- Reusable binding
- Action/Validator version

Secret **value**だけAttemptごとcurrent SecretsProviderから再materialize。

## 11. Descendant reactivation / reuse

### blocked/skipped Job

まだ成功実行していないためcurrent dependenciesで `if/with` を通常activationとして再評価可能。

### failed non-target Job

勝手にRetryしない。

### successful Job

`03` strict reuse:

- current `if=true`
- expected persistent Input/binding digest match
- upstream Artifact match
- Definition/executor/Validator identity match
- Payload/Artifact availability
- stored Output validation re-pass
- reuse eligible

Mismatch -> `successful_job_result_not_reusable`, new Workflow Run必要。

### Dynamic expansion

Committed `expanded` setは`05` expansion digest exact match必須。Mismatch=`dynamic_expansion_not_reusable`。

## 12. Internal timeout

`timeout-minutes`のcanonical intervalは`04`と同じ:

```text
start = internal Attempt claim transaction成功直後のRunner monotonic time
end   = Attempt terminal DB commit
```

Action本体だけではなく、Action Runner spawn/bootstrap/handshake、result transport/validation、Schema/Validator/success_if/SecretGuard/PayloadStore terminalizationまで含む。

Deadline到達時点でAttemptはtimeout outcomeへ固定する。

1. Runnerがdeadline到達を検知
2. timeout-expired flagを立てる
3. ChildがIPC可能なら`cancel_requested(reason=timeout)`を送る
4. ready前等でIPC不可ならChild terminate可
5. grace default10秒は**cleanup猶予**として待つ
6. grace中にresult/errorを受けても成功として採用しない
7. 未終了ならterminate
8. open Stepはincompleteへclose
9. Attempt=`failure/job_timeout/retryable=true`
10. Retry policy

Resultをdeadline前に受信しただけでは成功確定ではない。**Attempt terminal success commitがdeadline前なら**timeoutを適用せず、terminal commit前にdeadline到達した場合は`job_timeout`を優先する。

External/Human/Reusableに`timeout-minutes`無し。

## 13. Cancel / result race

Root Cancel transactionが`cancel_requested=1`をcommitした時点を境界とする。

- Action result terminal commitがCancelより先 -> そのJob resultは成立。Cancelは残りのRunへ適用
- Cancel commitが先 -> その後のrunning internal resultはcurrent successとして採用せずAttempt/Jobをcancelledへ収束
- Cancel後new activation禁止
- Runtime Handleのcleanup処理はAttempt terminal/fencingまで動作し得る

External/Human late submitは既存規則どおりreject。

Workflow最終conclusionはCancel requestが成立したRunでは`cancelled`。

## 14. Runner Lost / Runtime restart

Runner lost/old Runtime orphan running Attempt -> `runner_lost/retryable=true` terminal failure -> Retry policy。

Runtime起動:

1. new runtime_instance_id/Bootstrap
2. non-terminal Runs
3. running orphan
4. ready queued/pending snapshots
5. overdue Retry
6. External Lease
7. Review/Child
8. Dynamic expansion/reuse pending
9. downstream/conclusion
10. Retention/cleanup

Completed RunはRecoveryのみでreopenしない。

## 15. Pause

- running internal継続
- new internal/external claim/Dynamic expansion禁止
- existing submit/review/Child進行可
- Lease expiry継続
- due retry pending snapshotを保持、new AttemptはResume後
- ready Job Input snapshotを再評価しない
- admitted paused RunはConcurrency holderを維持する
- paused Concurrency waiterはslot無し、`concurrency_queued_at`を保持

## 16. Cancel

- cancel_requested
- queued/retry-wait/waiting executor cancel
- active Lease invalidate
- running internal cooperative cancel
- Child cancel
- new activation禁止
- Concurrency holder解放後waiting Runをwake

## 17. External / Human

Lease requeue=same AttemptでRetryではない。

Lease fail=Attempt failure -> Retry policy。

Human reject=retryable false defaultだがYAML retry.ifで明示Automatic Retry可能。Retry時はnew Review。

## 18. Recovery idempotency

Repeated RecoveryでAttempt/Task/Lease/Review/Dynamic/Child/retry schedule/reuse resultを重複生成しない。

Reopened Runが`wait_reason=concurrency`なら通常のConcurrency waitingとして復元し、pending Job Inputと保存済み`concurrency_queued_at`を再評価/再採番しない。

## 19. 受入条件

1. retry absent/default/backoff + `now >= retry_not_before`
2. overflow-safe saturating backoff for huge attempt number
3. delay always finite and <= max_seconds
4. core retryable map/parent override
5. pending snapshot exact copy all executors
6. Retry queued clears Job conclusion/completed_at
7. no new Attempt at retry scheduling/request
8. due executor-specific Attempt creation
9. External validation failure follows scheduler retry flow
10. Manual Retry Input/version eligibility
11. non-terminal Run retry does not increment run_attempt
12. completed Run retry increments run_attempt
13. completed Run concurrency reacquire queue/reject/available + fresh queue timestamp
14. reject leaves Run unchanged
15. reopen clears conclusion/completed/failure/current Workflow Output
16. target no with/item/version re-eval
17. Secret value re-materializes but binding fixed
18. descendant blocked/skipped reactivation
19. successful Job current-context reuse
20. Dynamic expansion strict reuse
21. timeout starts at claim and covers bootstrap through terminal commit
22. timeout result-before-deadline but commit-after-deadline discarded
23. cancel/result commit ordering
24. runner_lost/pause/cancel/recovery idempotency
