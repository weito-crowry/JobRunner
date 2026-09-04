# 07. External / Human Executor 詳細設計

- Status: Draft v0.5
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

Validatorはoptional。Default lease=60分、expiry=`requeue`。

## 3. External activation

1 transaction:

1. final Input/continue-on-error確定
2. validator ID/version snapshot確認
3. new Attempt
4. External Task
5. Job=`waiting_external`
6. Event

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
- Retry policy

Pause中もlease clockは進む。

### 5.1 expiry detection

外部clientの次回claim/submitを待たない。`03`のRuntime Maintenance Loopが`expires_at`をdeadlineとして検出し、期限到来時にcurrent Leaseをconditional transactionで処理する。

Runtime restart時はoverdue LeaseをRecoveryで先に処理する。

Expired/released/invalidated Lease submit reject。

## 6. `task_submit`

Request:

```text
task_id
lease_id
result                 # 任意JSON-compatible value
artifacts optional     # External Reference Artifact metadataのみ
claim_next optional
request_id optional
```

Result validation order:

1. JSON-compatible/canonical
2. optional `outputs.schema`
3. optional Custom Validator
4. optional `success_if`
5. SecretGuard（ExternalにSecret注入は無いがmetadata/result共通guard）
6. PayloadStore

### 6.1 Custom Validator

Validator Registryは親Runtime bootstrapで再構築済み。

External submit処理内でtrusted lightweight Validator callableを実行する。Validatorへ渡す:

```text
result value
persistent Job Input
```

Secret value/Runtime Handleは渡さない。

- valid=true -> 続行
- valid=false -> Attempt failure。Validator code/message/retryable/detailsをstructured failureへ
- exception -> `validator_exception`, retryable=false

Validation failureでもsubmitされたTask/Leaseは当該Attemptについてterminalにし、Retry policyが必要ならnew Attempt + new Taskを作る。Invalid resultを同じLeaseで再submitさせない。

重いvalidationはValidatorに入れずnormal internal Jobへ分離する。

### 6.2 Output persistence

<= threshold SQLite、> threshold filesystem blob。Hidden size failure無し。

### 6.3 Artifact

`task_submit.artifacts` は `storage_kind=external` の参照登録だけ。Coreはfetch/uploadしない。

### 6.4 Atomic submit

Logical operation:

- current Lease validation
- result/schema/validator/success_if
- PayloadStore prepare
- Artifact metadata
- Attempt/Job terminal
- Lease release / Task complete
- Event / idempotency result

Blob crash consistencyは`08`。

## 7. `claim_next`

Submit terminal処理成功後、別transactionで通常`task_claim`と同selection/authorizationを使用。Next claim失敗でsubmitをrollbackしない。

## 8. Pause / Cancel / Recovery

Pause:

- new claim禁止
- existing Lease submit受理
- Maintenance Loopによるlease expiry継続

Cancel:

- available Task cancelled
- active Lease invalidated
- Attempt/Job cancelled
- late submit reject

Recovery:

- active unexpired Lease維持
- expiredへpolicy
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

Activation: new Attempt + one pending Review + Job=`waiting_review`。

Outcome:

```text
approve|reject
```

Approve -> success。
Reject -> failure `human_rejected`, retryable=false default。

Human lease/timeout無し。Concurrent submit first-wins。

標準Job Outputは `null`。Review metadataは`wf_review_info`。

## 10. Human Pause / Cancel / Retry

Pause中review submit受理。

Cancel後late submit reject。

Reject後Retryはnew Attempt + new pending Review。

## 11. Idempotency / Authorization

`task_claim/task_submit/review_submit` optional request_id。Scope/TTLは`08/11`。

All read/write Authorization。

## 12. 受入条件

1. External arbitrary JSON result
2. External Custom Validator valid/invalid/exception/retryable
3. large result spill
4. External Artifact reference only
5. concurrent claim exactly one
6. claim/claim_next same ordering
7. lease expiry without client traffic
8. paused lease expiry
9. lease requeue/fail
10. stale/cancel submit reject
11. Recovery overdue lease
12. Human output null / first-wins / retry
13. idempotency/authorization
