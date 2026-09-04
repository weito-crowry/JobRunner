# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 基本原則

1. Retryはnew Attempt。
2. failed Attemptをqueuedへ戻さない。
3. Retry Input/Definition/Action versionは元Job/Run snapshotを使う。
4. automatic retryは明示時のみ。
5. manual retryはService API。
6. execution timeoutはinternal Jobのみ、未指定なら無期限。
7. Cancelはgraceful標準。
8. Recoveryは履歴を壊さず、manual retry以外でcompleted Runをreopenしない。

## 2. Structured Failure

```text
category
code
message
retryable
details optional
occurred_at
```

代表code:

```text
action_exception
action_process_exit
output_validation_failed
ipc_protocol_error
job_timeout
runner_lost
action_version_mismatch
external_lease_expired
human_rejected
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

未指定:

```text
max-attempts = 1
automatic retry disabled
```

判定:

1. Attempt failure確定
2. cancel確認
3. max-attempts確認
4. retry `if`
5. backoff算出
6. Jobをretry待機へ (`retry_not_before`)
7. Event

Attemptはbackoff終了後、実際に再実行開始するとき作る。

## 4. Manual Retry

MVP public操作は failed Job指定。

対象Workflow Runが `completed/failure` の場合、explicit manual retry transactionで同じRunをreopenする。

- `run_attempt += 1`
- target Jobをretry待機
- failure由来 `blocked` descendantsを再評価可能にする
- 過去Attempt/Eventは保持

`success/cancelled` Workflow Runはmanual retry対象外。

## 5. Retry固定値

変えない:

- Workflow Definition snapshot
- Workflow Input snapshot
- Job final Input snapshot
- Dynamic item/iteration/order
- Reusable binding
- Action ID/version
- `continue-on-error` snapshot

Secret referenceは固定するが、Secret valueは各internal Attempt起動直前に再materialize。

## 6. Internal Job timeout

`timeout-minutes` は **internal executorだけ**。

期限到達:

1. Runner -> Action Runner cancel request
2. configurable graceful grace period
3. 終了しなければAction子Process terminate
4. Attempt failure `job_timeout`
5. Retry policy

Runner Process自身はkillしない。

ExternalはLease、Humanは無期限Review待ち、ReusableはChild内Job timeoutで制御する。

## 7. Runner Lost

Heartbeat/liveness判定でRunner lost確定後、そのRunner current running Attemptだけをconditional updateで `runner_lost` failureへ。

その後Retry policy。

Old runtime/runner instanceからのlate updateはfencingで拒否。

## 8. Runtime restart Recovery

起動時:

1. new `runtime_instance_id`
2. Registry/Pool再構築
3. non-terminal Runs列挙
4. running internal Attempt ownership回復/runner_lost確定
5. active External Lease維持、expiredへpolicy
6. waiting_review維持
7. retry backoff維持
8. Dynamic expansion/Reusable relationをidempotent復元
9. downstream activation/conclusion再評価

### 8.1 terminal invariant

`completed` Workflow RunはRecoveryではreopenしない。

唯一のreopen経路は認可済みexplicit manual retry。

## 9. Pause

Workflow Run単位。

- running internal継続
- new internal claim禁止
- new Dynamic expansion禁止
- new External claim禁止
- existing External submit受理
- pending Human submit受理
- started Child Workflowは継続

ResumeでScheduling再開。new Attemptは作らない。

## 10. Cancel

Workflow cancel:

- cancel_requested=true
- queued/retry-wait Jobs -> cancelled
- waiting External -> Task/Lease invalidated + Attempt/Job cancelled
- waiting Human -> Review/Attempt/Job cancelled
- running internal -> cancel request
- current Child Workflowへ伝播
- new activation/expansion禁止

Workflow conclusionは `cancelled`。

## 11. External Lease expiry

- `requeue`: same Attempt、Task availableへ。Retryではない。
- `fail`: Attempt failure `external_lease_expired` -> Retry policy。

## 12. Human Reject

`human_rejected` failure。人間操作でsuccessへ書き換えない。必要ならmanual retry。

## 13. Recovery idempotency

Repeated Recoveryで以下を重複生成しない:

- Attempt
- External Task
- Human Review
- Dynamic expansion/jobs
- Child Workflow Run
- Event（同じstate transition）

状態条件 + unique constraintを使う。

## 14. Workflow conclusion

required Jobにnon-allowed failure/blockedがあればfailure。

`continue-on-error` failureはJob自身のfailureを保持したままWorkflow failure寄与を抑制。

Cancel request由来はcancelled。

## 15. 受入条件

1. auto retry disabled default
2. max-attempts/backoff/retry if
3. manual retry completed failure reopen
4. success/cancelled Run retry拒否
5. internal timeout + retry
6. external/human/reusable timeout無し
7. runner_lost
8. runtime restart terminal invariant
9. pause/resume
10. cancel all executor types
11. recovery duplicate防止
