# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.6
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `04`, `07`, `08`, `11`

## 1. 基本原則

1. Retryはnew Attempt。
2. Failed Attemptをqueuedへ戻さない。
3. Retry Input/Definition/Action/Validator version/Dynamic iteration/Reusable bindingは固定。
4. Automatic Retryは`retry` block明示時のみ。
5. Manual Retryはfailed Job指定。
6. Retryには既存Attemptのpersistent Input snapshotが必須。
7. Internal timeoutのみ。未指定無期限。
8. Cancelはgraceful。
9. Recoveryだけでcompleted Runをreopenしない。
10. Manual Retry後のsuccessful descendantは`03`のsame-run Result Reuse必須。

## 2. Structured Failure

```text
category
code
message
retryable
details optional
occurred_at
```

`retryable`は必須boolean。Failure producerが必ず明示し、missingを暗黙trueにしない。

## 3. Failure retryability defaults

Core生成failureの既定:

| code | retryable | 理由 |
| --- | --- | --- |
| `runner_lost` | true | 実行基盤の一時的消失 |
| `job_timeout` | true | 同Input再試行で回復可能性あり |
| `external_lease_expired` (`fail` policy) | true | 外部owner不在/遅延 |
| `payload_storage_failed` | true | local I/O一時障害の可能性 |
| `action_process_exit` | false | Action crashは既定で明示Retryさせる |
| `action_exception` | false | 未分類Action例外 |
| `ipc_protocol_error` | false | 実装/契約不整合の疑い |
| `result_file_invalid` | false | 実装/破損 |
| `output_validation_failed` | false | 同Output再試行で直る保証無し |
| `validator_exception` | false | Validator実装問題 |
| `domain_validation_failed` | Validator結果 | 親Validatorが判断 |
| `success_condition_failed` | false | 業務条件不成立 |
| `action_version_mismatch` | false | Registry/Run snapshot不整合 |
| `validator_version_mismatch` | false | Registry/Run snapshot不整合 |
| `human_rejected` | false | 人間判断 |
| `secret_not_found` | false | 環境設定修正が必要 |
| `secret_value_invalid` | false | Provider contract違反 |
| `secret_value_persistence_blocked` | false | Security violation |
| `successful_job_result_not_reusable` | false | new Runが必要 |
| `retry_input_unavailable` | false | new Runが必要 |
| `cancelled` | false | Retry対象failureではない |
| `internal_error` | false | 分類不能Core error |

親`ActionFailure`は指定された`retryable`をそのまま使用する。

`retryable=false`でもYAMLで `retry.if: ${{ true }}` と明示すればAutomatic Retryを許可する。`failure.retryable`は安全な既定判断であり絶対禁止フラグではない。ただし `retry_input_unavailable` はAttemptが無いためAutomatic/Manual Retryの対象外。

## 4. Retry YAML canonical schema

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

### 4.1 `retry` block absent

```text
automatic retry = disabled
max attempts = 1
```

### 4.2 `retry` block present

Canonical defaults:

```text
max-attempts = 2
if = ${{ failure.retryable }}
backoff.initial-seconds = 0
backoff.max-seconds = initial-seconds
backoff.multiplier = 1.0
```

つまり `retry: {}` は「最大2 Attempt、retryable failureを1回だけ即時再試行」。

Validation:

- `max-attempts`: integer `2..9223372036854775807`
- `if`: CEL boolean expression
- `initial-seconds`: finite number >=0
- `max-seconds`: finite number >= initial-seconds
- `multiplier`: finite number >=1.0
- unknown key reject

`max-attempts=1`でAutomatic Retryを無効化したい場合は`retry` block自体を省略する。意味を二重化しない。

### 4.3 Backoff calculation

次Attempt番号を `n`（2から開始）として:

```text
delay = min(max_seconds,
            initial_seconds * multiplier ** (n - 2))
```

initial=0なら常に0。JitterはMVP無し。Delay計算結果はfiniteでなければDefinition validation errorとなる範囲へ事前制約する。

## 5. Automatic Retry flow

Automatic Retryはfailed Attemptに対してのみ。

1. Attempt failure確定
2. Workflow cancel確認
3. current attempt_no < max-attempts確認
4. Retry `if`を`failure` contextで評価
5. false -> Job failure terminal
6. true -> backoff計算
7. Jobをqueued/retry waitingへし `retry_not_before` 保存
8. Maintenance Loopへdeadline notify
9. Event `retry_scheduled`

Backoff予約時new Attemptを作らない。Deadline到来後、Runner/internalまたはexecutor activation時にnew Attemptを作る。

Retry `if` expression errorはretryせず、元failureを保持したまま `retry_condition_evaluation_failed` diagnostic/Eventを追加する。元failureを別failureへ上書きしない。

## 6. Manual Retry eligibility

`wf_retry(workflow_run_id, job_run_id)`。

必須:

- target Job conclusion=`failure`
- terminal failed Attempt >=1
- latest failed Attemptにpersistent `input_json`
- Workflow Run conclusion=`failure` またはnon-terminal
- snapshotted Action/Validator/Reusable binding versionをcurrent Registryで提供可能

Reject:

- success/cancelled Run
- Dynamic expansion自体のfailureでAttempt無し
- activation/Input resolutionでAttempt開始前failure

Input snapshot無し:

```text
code=retry_input_unavailable
retryable=false
```

Version不足は`action_version_mismatch|validator_version_mismatch`。

## 7. Manual Retry reopen

Completed/failure Run:

1. eligibility + idempotency
2. `run_attempt += 1`
3. Run reopen
4. target Job retry waiting/queued (`retry_not_before=null`)
5. blocked/skipped descendants activation再評価
6. successful descendants `reuse_check_pending=1`
7. Event

過去Attempt/Output/Artifact/Eventは削除しない。

Manual Retryはautomatic `max-attempts`上限をリセットしない。Manual Retryによるnew Attemptは既存`attempt_no+1`を採番し、そのAttempt failure後のautomatic Retry可否は **その時点のattempt_noとmax-attemptsを比較**する。したがって既にmax-attempts以上なら追加Automatic Retryは起こらない。

## 8. Retry固定Input

Retryは基準failed Attemptのpersistent `input_json`をnew Attemptへcopy。

再評価しない:

- `with`
- current upstream Output/state
- Dynamic item/iteration/order
- continue-on-error
- Reusable binding
- Action/Validator version

Secret referenceは固定、materialized valueはAttemptごと再解決。

## 9. Descendant reactivation / reuse

Blocked/skipped descendantはcurrent dependenciesで`if/with`再評価。

Failed non-target descendantは自動Retryしない。

Successful descendantは`03` reuse check:

- match + eligible + Payload integrity + Action/Validator version available -> success reuse
- otherwise `successful_job_result_not_reusable`, retryable=false, new Workflow Run必要

## 10. Internal timeout

1. timeout cancel
2. grace default10秒
3. child terminate
4. `job_timeout/retryable=true`
5. Retry policy

External/Human/Reusableに`timeout-minutes`無し。

## 11. Runner Lost / Runtime restart

Runner lost/old-runtime orphan running Attempt:

```text
runner_lost
retryable=true
```

でconditional terminal化してRetry policyへ。

Runtime起動時:

1. new runtime_instance_id
2. Registry/Pool/Storage
3. non-terminal Runs
4. running orphan
5. overdue Retry deadlines
6. active/expired External Leases
7. pending Review
8. waiting Child
9. Dynamic expansion
10. reuse_check_pending
11. downstream/conclusion

Completed RunはRecoveryでreopenしない。

## 12. Pause

- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human submit/started Child進行可
- Lease expiry処理継続
- retry_not_before到来は保持するがnew Attempt開始はResume後

## 13. Cancel

- cancel_requested
- queued/retry-wait/waiting executor cancel
- active External Lease invalidated
- running internal cancel
- Child cancel propagation
- new activation禁止

Workflow conclusion cancelled。

## 14. External / Human

Lease `requeue`: same Attempt、Retryではない。

Lease `fail`: `external_lease_expired/retryable=true` -> Retry policy。

Human reject: `human_rejected/retryable=false`。Manual Retryはpersistent Inputがあるため可能。

## 15. Recovery idempotency

Repeated RecoveryでAttempt/Task/Lease/Review/Dynamic/Child/reuse/retry scheduleを重複生成しない。

## 16. 受入条件

1. retry block absent disabled
2. `retry:{}` exact default = max2 / retryable / zero backoff
3. retry numeric validation/backoff formula
4. failure retryable mapping
5. ActionFailure retryability propagation
6. auto retry failed Attemptのみ
7. Maintenance deadline wake
8. Retry condition error preserves original failure
9. manual retry Input/version eligibility
10. manual retry attempt numbering/max-attempt interaction
11. descendant reactivation/reuse
12. internal timeout/runner_lost defaults
13. pause/cancel/recovery idempotency
