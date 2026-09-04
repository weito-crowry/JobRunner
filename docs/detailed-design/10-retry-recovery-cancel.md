# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `04`, `05`, `07`, `08`, `11`

## 1. 基本原則

1. Retry=new Attempt。
2. Failed Attempt rowを書き換えない。
3. Retry Input/Secret bindings/Definition/Action/Validator/Dynamic iteration/Reusable binding固定。
4. Automatic Retryは`retry` block明示時のみ。
5. Manual Retryはfailed Job指定。
6. Retryにはfailed Attempt persistent Input snapshot必須。
7. Internal timeoutのみ。未指定無期限。
8. Cancel=graceful。
9. Recoveryだけでcompleted Run reopen無し。
10. Manual Retry後successful descendantは`03/05` strict reuse必須。

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

Delay for next Attempt `n>=2`:

```text
min(max_seconds, initial_seconds * multiplier ** (n-2))
```

Jitter無し。

## 5. Retry pending snapshot

Retry request/schedule時に基準failed Attemptから:

```text
input_json
secret_bindings_json
input_digest
```

をJob `pending_*`へexact copyする。

さらに:

```text
ready_at = retry schedule/request time
retry_not_before = due time or NULL
status = queued
```

とする。

**new Attemptはまだ作らない。**

Due後:

- internal -> Runner claimがpending snapshotをnew Attemptへcopy
- external -> scheduler activationがnew Attempt + Taskへcopy
- human -> new Attempt + Reviewへcopy
- reusable -> new Attempt + Child Runへcopy

Attempt作成後pending fieldsはclear可能。

この方式で全executorのRetry Inputを同じ規則にする。

## 6. Automatic Retry flow

Failed Attempt terminal時:

1. Workflow cancel確認
2. current attempt_no < max-attempts
3. Retry `if`を`failure` contextで評価
4. false -> Job completed/failure
5. true -> delay計算
6. §5 pending snapshot作成
7. `retry_not_before`保存
8. Maintenance notify
9. `retry_scheduled` Event/Execution Log

Retry condition errorはRetryせず元failure維持 + diagnostic/Event。

Validation/domain failureをExternal submitで受けた場合もこのflow。Submit transaction内でnew Attempt/Taskは作らない。

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

Attempt/Input無し:

```text
retry_input_unavailable
retryable=false
```

## 8. Manual Retry reopen

Completed/failure Run:

1. eligibility/idempotency
2. `run_attempt += 1`
3. Run reopen
4. target Jobへ§5 pending snapshot、`retry_not_before=NULL`
5. blocked/skipped descendants activation再評価対象へ戻す
6. whole-skipped Dynamic expansion rowは`05`規則でreset可能
7. successful descendants `reuse_check_pending=1`
8. successful Dynamic group/expanded rows `05` reuse check対象
9. Event/Execution Log

Past Attempt/Output/Artifact/Event削除無し。

Manual Retryはautomatic max-attempts budgetをリセットしない。New Attempt番号=existing max+1。その失敗後automatic Retry可否はactual attempt_noとmax-attempts比較。

## 9. Target Retry固定値

Target Retryで再評価しない:

- `with`
- upstream Output/state
- Secret binding name/path
- Dynamic item/iteration/key/order
- continue-on-error
- Reusable binding
- Action/Validator version

Secret **value**だけAttemptごとcurrent SecretsProviderから再materialize。

## 10. Descendant reactivation / reuse

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
- eligible

Mismatch -> `successful_job_result_not_reusable`, new Workflow Run必要。

### Dynamic expansion

Committed `expanded` setは`05` expansion digest exact match必須。Mismatch=`dynamic_expansion_not_reusable`。Same Runでgenerated setを差し替えない。

## 11. Internal timeout

1. timeout cancel
2. grace default10秒
3. terminate
4. `job_timeout/retryable=true`
5. Retry policy

External/Human/Reusableに`timeout-minutes`無し。

## 12. Runner Lost / Runtime restart

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

## 13. Pause

- running internal継続
- new internal/external claim/Dynamic expansion禁止
- existing submit/review/Child進行可
- Lease expiry継続
- due retry pending snapshotを保持、new AttemptはResume後
- ready Job Input snapshotを再評価しない

## 14. Cancel

- cancel_requested
- queued/retry-wait/waiting executor cancel
- active Lease invalidate
- running internal cooperative cancel
- Child cancel
- new activation禁止

Workflow conclusion=cancelled。

## 15. External / Human

Lease requeue=same AttemptでRetryではない。

Lease fail=Attempt failure -> Retry policy。

Human reject=retryable false defaultだがYAML retry.ifで明示Automatic Retry可能。Retry時はnew Review。

## 16. Recovery idempotency

Repeated RecoveryでAttempt/Task/Lease/Review/Dynamic/Child/retry schedule/reuse resultを重複生成しない。

## 17. 受入条件

1. retry absent/default/backoff
2. core retryable map/parent override
3. pending snapshot exact copy all executors
4. no new Attempt at retry scheduling/request
5. due executor-specific Attempt creation
6. External validation failure follows scheduler retry flow
7. Manual Retry Input/version eligibility
8. Manual run_attempt/attempt numbering
9. Target no with/item/version re-eval
10. Secret value re-materializes but binding fixed
11. descendant blocked/skipped reactivation
12. successful Job current-context reuse
13. Dynamic expansion strict reuse
14. timeout/runner_lost
15. pause/cancel/recovery idempotency
