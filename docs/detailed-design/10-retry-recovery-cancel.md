# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03`, `04`, `07`, `08`

## 1. 基本原則

1. Retryはnew Attempt。
2. Failed Attemptをqueuedへ戻さない。
3. Retry Input/Definition/Action version/Dynamic iteration/Reusable bindingは固定。
4. Automatic retryは明示時のみ。
5. Manual retryはfailed Job指定。
6. Internal timeoutのみ。未指定無期限。
7. Cancelはgraceful。
8. Recoveryだけでcompleted Runをreopenしない。
9. Manual Retry後の成功済みdescendantは`03`のsame-run Result Reuse判定を必須とする。

## 2. Structured Failure

```text
category
code
message
retryable
details optional
occurred_at
```

代表:

```text
action_exception
action_process_exit
output_validation_failed
ipc_protocol_error
result_file_invalid
payload_storage_failed
job_timeout
runner_lost
action_version_mismatch
external_lease_expired
human_rejected
secret_value_persistence_blocked
successful_job_result_not_reusable
cancelled
internal_error
```

## 3. Automatic Retry

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

未指定: max-attempts=1 / auto retry disabled。

Failure後:

1. cancel確認
2. attempt残数
3. retry if
4. backoff
5. Job retry waiting (`retry_not_before`)
6. Event

Backoff予約時Attempt rowは作らない。

## 4. Manual Retry

`wf_retry(workflow_run_id, job_run_id)`。

条件:

- target Job conclusion=`failure`
- Workflow Run conclusion=`failure` またはまだnon-terminal
- success/cancelled Runはretry不可

Completed/failure Runならexplicit transactionで:

1. Run reopen、`run_attempt += 1`
2. target Job retry waiting
3. target dependency closureのblocked/skipped descendantsをactivation再評価状態へ
4. successful descendantsへ`reuse_check_pending=1`
5. Event/idempotency

過去Attempt/Output/Artifact/Eventは削除しない。

## 5. Retry固定Input

Retryは元persistent Job Inputを使う。Current upstream Output/stateを再評価してInputを書き換えない。

Secret referenceは固定、materialized valueはAttemptごと再解決。

Target Retryのreuse contextも元execution contextを維持する。

## 6. Descendant reactivation

Blocked/skipped descendantはAttemptを持たない場合が多いため、current dependenciesで`if/with`を再評価できる。

Explicit condition falseで以前skippedだったJobも、dependencyが変わったため再評価対象にしてよい。

Failed descendantはtarget指定されない限り自動Retryしない。

## 7. Successful descendant Result Reuse

Manual Retry後、成功済みJobを無条件に使わない。

Dependencies terminal後に`03`の現在reuse contextを計算:

- key一致 + eligible + Payload存在/integrity OK + Action version available -> existing success reuse
- それ以外 -> `successful_job_result_not_reusable`

MVPではsuccess Jobを新Inputで自動再実行しない。新Workflow Runを要求する。

Reuse mismatch自体をautomatic/manual retryで回復しない (`retryable=false`)。

## 8. Internal timeout

期限到達:

1. childへtimeout cancel
2. grace default10秒
3. child terminate
4. `job_timeout`
5. Retry policy

External/Human/Reusableに`timeout-minutes`無し。

## 9. Runner Lost

Heartbeat/liveness timeout、またはParent restart後old runtime orphan running Attemptを `runner_lost` failureへconditional transition。

Retry policy適用。

旧Runtime/Runner late update reject。

## 10. Runtime restart Recovery

1. new runtime_instance_id
2. Registry/Pool/Storage初期化
3. non-terminal Runs
4. running internal orphan/Runner state
5. active/expired External Lease
6. pending Review
7. waiting Child
8. retry backoff
9. Dynamic expansion
10. `reuse_check_pending`
11. downstream/conclusion

Completed RunはRecoveryでreopenしない。

## 11. Pause

- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human submit/started Child進行可
- reuse pending checkはpure state validationなので実行可能だが、結果としてnew Job activationはResumeまで行わない

## 12. Cancel

- cancel_requested
- queued/retry-wait/waiting executorをcancel
- active External Lease invalidated
- running internal cancel
- Child cancel propagation
- new activation禁止

Workflow conclusion cancelled。

## 13. External Lease / Human

Lease `requeue`: same Attempt、Retryでない。

Lease `fail`: `external_lease_expired` -> Retry policy。

Human reject: `human_rejected`。Direct success override無し。

## 14. Recovery idempotency

Repeated Recoveryで重複しない:

```text
Attempt
Task/Lease
Review
Dynamic expansion/jobs
Child Run
reuse Event/state transition
```

State条件 + unique constraint。

## 15. 受入条件

1. auto retry/backoff
2. manual retry reopen/run_attempt
3. target Input fixed
4. blocked/skipped descendant reevaluate
5. failed descendant no-auto-retry
6. success descendant reuse match
7. mismatch/ineligible/payload missing -> new Run required
8. internal timeout
9. parent restart runner_lost
10. terminal invariant
11. pause/cancel
12. recovery idempotency
