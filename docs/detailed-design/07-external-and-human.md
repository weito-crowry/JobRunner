# 07. External / Human Executor 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `11`, `12`

## 1. 共通原則

1. External/Human JobはRunnerを占有しない。
2. `action/uses/runs-on/timeout-minutes`は禁止。
3. Secret参照は禁止。
4. Job Inputはactivation時snapshot。
5. activation時にAttempt作成。
6. Retryはnew Attempt。
7. state changeはService/Authorization/idempotency経路。
8. failed/cancelled Jobを管理操作で直接successへ書き換えない。
9. YAML未定義manual skip操作無し。

## 2. External LLM

```yaml
jobs:
  classify:
    executor: external_llm
    validator: domain.validate_classification
    with:
      text: ${{ inputs.text }}
    external:
      lease-minutes: 120
      on-lease-expiry: requeue
```

Effective config:

```text
Job external value > Workflow effective settings > System config > canonical default
```

Canonical default:

```text
lease-minutes=60
on-lease-expiry=requeue
```

Effective `lease-minutes/on-lease-expiry` はAttempt/Task activation時にsnapshotし、当該Taskのrequeueでも変えない。Retryでnew Attempt/Taskを作る場合も同じWorkflow Run Definition/effective settings snapshotから解決する。

## 3. External activation

1 transaction:

1. `02` persistent Input snapshot確定
2. Validator ID/version確認
3. effective lease config確定
4. new Attempt
5. External Task
6. Job=`waiting_external`
7. Attempt Execution Log作成
8. Event

One Task / Attempt。

## 4. Task / Lease

Task:

```text
available|leased|completed|cancelled
```

Lease:

```text
active|expired|released|invalidated
```

`task_claim` candidate順:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source order ASC
5. ready_at ASC
6. Job Run ID

Atomic exactly-one claim。

Claim時 `expires_at = claim_time + Task snapshot lease duration`。

### 4.1 Lease feature boundary

MVP Leaseは固定expiryの単純ownership window。

無し:

```text
lease heartbeat
lease renew / extend
lease ownership transfer
```

長い処理は十分な`lease-minutes`をDefinition/Systemで指定する。

## 5. Lease expiry

`requeue`:

- same Attempt
- current Lease -> expired
- Task -> available
- new Attempt無し

`fail`:

- Lease -> expired
- Task terminal
- Attempt failure `external_lease_expired`
- `10` Retry policyへ

Pause中もlease clockは進む。

Maintenance Loopがclient traffic無しでexpiryを処理。Restart時overdue Leaseを先処理。

Expired/released/invalidated Lease submit reject。

## 6. `task_submit`

Request:

```text
task_id
lease_id
result
artifacts optional
claim_next optional
request_id optional
```

Result validation:

1. JSON-compatible/canonical
2. optional `outputs.schema`
3. optional Custom Validator
4. optional `success_if`
5. SecretGuard
6. PayloadStore prepare

### 6.1 Custom Validator

Validator receives:

```text
result
persistent Job Input
```

Secret/Runtime Handle無し。

- valid -> continue
- invalid -> Attempt failure with Validator structured failure
- exception -> `validator_exception`, retryable=false

Validation/domain failureでもsubmitは当該Attemptについてterminalに受理する。Retry可能なら`10`がJobをretry waitingへscheduleする。

**同じsubmit transaction内でnew Attempt/new Taskは作らない。** Backoff到来後またはimmediate due activationでnew Attempt/Taskを作る。

Invalid resultを同じLeaseで再submitさせない。

### 6.2 Output persistence

Effective threshold以下SQLite、超過PayloadStore blob。Hidden output max無し。

### 6.3 Artifact

`task_submit.artifacts` はExternal Reference Artifact metadataのみ。Core binary upload/fetch無し。

## 7. Atomic submit + `claim_next`

`claim_next=true` のresponseをidempotent replay可能にするため、**submit terminal DB state + optional next Task claim + completed idempotency resultを同一SQLite write transactionで確定する。**

Long validation/filesystem prepareはtransaction前に行う。

正規flow:

1. preliminary Lease/Task state read
2. result/schema/Validator/success_if/SecretGuard
3. PayloadStore temp/final prepare
4. `BEGIN IMMEDIATE`
5. idempotency key再確認
6. current Task/Lease owner + `expires_at`を再確認
7. Artifact metadata準備
8. Attempt/Job terminal or Retry schedule
9. Lease released / Task completed
10. downstream terminal bookkeepingのうち同transactionで必要なDB stateを確定
11. `claim_next=true`なら同じAuthorization + `task_claim` orderingでavailable Taskを検索
12. candidateあり -> 同transactionでLease作成/Task leased
13. candidate無し -> `next_task=null`
14. submit response全体（next_task含む）をidempotency recordへ保存
15. commit

`claim_next`に候補が無いことはsubmit failureではない。

Internal DB/storage errorでtransaction自体をcommitできない場合はsubmitも未commitなのでcaller retry可能。Prepared orphanは`08` cleanup対象。

Idempotency replayは保存済みresponseを返し、別Taskを追加claimしない。

## 8. Pause / Cancel / Recovery

Pause:

- new claim禁止
- existing Lease submit受理
- lease expiry継続

Cancel:

- available Task cancelled
- active Lease invalidated
- Attempt/Job cancelled
- late submit reject

Recovery:

- active unexpired Lease維持
- expired -> snapshotted policy
- terminal Attemptへnew Lease不可

## 9. Human Review

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs }}
```

Humanは`action/validator/uses/runs-on/success_if/external/timeout-minutes/secrets`禁止。

Activation:

- persistent Input snapshot
- new Attempt
- one pending Review
- Attempt Execution Log
- Job=`waiting_review`

Outcome:

```text
approve|reject
```

Approve -> success。
Reject -> failure `human_rejected`, retryable=false default。

Human lease/timeout無し。Concurrent submit first-wins。Job Output=`null`。

### 9.1 Human control boundary

許可state change=pending Reviewへのapprove/rejectのみ。

無し:

```text
failed/cancelled Job -> success override
manual Job skip
completed Review rewrite
```

Reject後は`wf_retry`でnew Attempt + new Review。

## 10. Human Pause / Cancel / Retry

Pause中review submit受理。Cancel後late submit reject。Reject後Retry=new pending Review。

## 11. Common Execution Log

External/Humanもinternalと同じAttempt Execution Logを持つ。

ExternalではRuntime/Serviceが少なくとも:

```text
task available
claim/lease
lease expiry/requeue
submit accepted/rejected
validation result
Attempt terminal
```

を記録する。

Humanでは:

```text
review pending
approve/reject
cancel/retry
Attempt terminal
```

を記録する。

CoreはExternal LLMだけ特別に「ログ禁止」としない。ただし全executor共通方針としてInput/Output全bodyをExecution Logへ自動複製せず、Input/Output APIをSource of Truthとする。SecretGuard/Authorizationは共通適用。

## 12. Idempotency / Authorization

`task_claim/task_submit/review_submit` optional request_id。Scope/hash/TTLは`08/11`。

All read/write Authorization。

## 13. 受入条件

1. Job>Workflow>System lease config hierarchy
2. Task config snapshot/requeue固定
3. External arbitrary JSON result/spill
4. External Validator valid/invalid/exception
5. Validator failure schedules Retry but no inline new Attempt
6. External Artifact reference only
7. concurrent claim exactly one
8. claim/claim_next same ordering
9. submit + next claim + idempotency one transaction
10. submit crash/replay does not double claim
11. no lease heartbeat/renew
12. lease requeue/fail/Pause/Recovery
13. stale/cancel submit reject
14. Human output null / first-wins / retry
15. Human override/skip/rewrite無し
16. External/Human common Execution Log
17. idempotency/authorization
