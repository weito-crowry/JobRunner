# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `03`, `04`, `07`, `08`, `11`

## 1. 基本原則

1. Retryはnew Attempt。
2. Failed Attemptをqueuedへ戻さない。
3. Retry Input/Definition/Action version/Dynamic iteration/Reusable bindingは固定。
4. Automatic retryは明示時のみ。
5. Manual retryはfailed Job指定。
6. **Retryには既存Attemptのpersistent Input snapshotが必須。**
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
retry_input_unavailable
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

Automatic Retryは**failed Attempt**に対してのみ判定する。Attempt開始前のactivation/expansion failureには適用しない。

Failed Attempt後:

1. cancel確認
2. attempt残数
3. retry if
4. backoff
5. Job retry waiting (`retry_not_before`)
6. Event

Backoff予約時new Attempt rowは作らない。

## 4. Manual Retry eligibility

`wf_retry(workflow_run_id, job_run_id)`。

必須条件:

- target Job conclusion=`failure`
- targetに少なくとも1件のterminal Attemptがある
- Retry基準となる最新failed Attemptに`input_json` persistent snapshotがある
- Workflow Run conclusion=`failure` またはまだnon-terminal

以下はreject:

- success/cancelled Workflow Run
- Dynamic expansionそのもののfailureで実Job Attemptが無い
- Job activation/Input resolutionがAttempt開始前にfailureしInput snapshotが無い

Input snapshot無し:

```text
code = retry_input_unavailable
retryable = false
```

この場合はDefinition/Input/外部条件を修正して**new Workflow Run**を開始する。

## 5. Manual Retry reopen

Completed/failure Runならexplicit transactionで:

1. Run reopen、`run_attempt += 1`
2. target Job retry waiting
3. target dependency closureのblocked/skipped descendantsをactivation再評価状態へ
4. successful descendantsへ`reuse_check_pending=1`
5. Event/idempotency

過去Attempt/Output/Artifact/Eventは削除しない。

## 6. Retry固定Input

Retryは基準failed Attemptのpersistent `input_json`をnew Attemptへcopyする。

Current upstream Output/state/item/`with` expressionを再評価してJob Inputを書き換えない。

固定:

- persistent Input
- Workflow Definition snapshot
- Action ID/version
- Dynamic item/iteration/order
- Reusable binding
- snapshotted continue-on-error

Secret referenceはInput内で固定、materialized Secret valueはAttemptごと再解決。

## 7. Descendant reactivation

Blocked/skipped descendantはAttemptを持たない場合が多いため、current dependenciesで`if/with`を再評価できる。

Explicit condition falseで以前skippedだったJobも再評価対象。

Failed descendantはtarget指定されない限り自動Retryしない。

## 8. Successful descendant Result Reuse

Manual Retry後、success Jobを無条件使用しない。

Dependencies terminal後に`03`の現在reuse contextを計算:

- key一致 + eligible + Payload integrity + Action version available -> existing success reuse
- otherwise -> `successful_job_result_not_reusable`

MVPではsuccess Jobをnew Inputで自動再実行しない。New Workflow Runを要求する。

Mismatchはretryable=false。

## 9. Internal timeout

1. childへtimeout cancel
2. grace default10秒
3. child terminate
4. `job_timeout`
5. Retry policy

External/Human/Reusableに`timeout-minutes`無し。

## 10. Runner Lost / Runtime restart

Runner lost、またはParent restart後old runtime orphan running Attemptを`runner_lost` failureへconditional transitionしRetry policyへ。

Runtime起動時:

1. new runtime_instance_id
2. Registry/Pool/Storage
3. non-terminal Runs
4. running internal orphan
5. External Lease
6. pending Review
7. waiting Child
8. retry backoff
9. Dynamic expansion
10. reuse_check_pending
11. downstream/conclusion

Completed RunはRecoveryでreopenしない。

## 11. Pause

- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human submit/started Child進行可
- pure reuse validation可。ただしnew activationはResume待ち

## 12. Cancel

- cancel_requested
- queued/retry-wait/waiting executor cancel
- active External Lease invalidated
- running internal cancel
- Child cancel propagation
- new activation禁止

Workflow conclusion cancelled。

## 13. External / Human

Lease `requeue`: same Attempt、Retryではない。

Lease `fail`: failed Attempt -> Retry policy。

Human reject: failed Attempt、persistent inputありなのでManual Retry可能。

## 14. Recovery idempotency

Repeated RecoveryでAttempt/Task/Lease/Review/Dynamic/Child/reuse transitionを重複生成しない。

## 15. 受入条件

1. auto retry failed Attemptのみ
2. manual retry requires persistent Input snapshot
3. pre-Attempt failure -> retry_input_unavailable/new Run
4. manual retry reopen/run_attempt
5. target Input exact copy
6. blocked/skipped descendant reevaluate
7. failed descendant no-auto-retry
8. success descendant reuse match/mismatch
9. internal timeout
10. parent restart runner_lost
11. terminal invariant
12. pause/cancel/recovery idempotency
