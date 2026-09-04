# 07. External / Human Executor 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`, `10-retry-recovery-cancel.md`

## 1. 目的

`external_llm` / `human` executor の activation、claim/lease/submit、review、Cancel、Retry、Recoveryを定義する。

## 2. 共通原則

1. External/Human JobはRunnerを占有しない。
2. `action/uses/runs-on/timeout-minutes`は禁止。
3. `${{ secrets.* }}`は禁止。
4. Job Inputはactivation時snapshot。
5. activation時にAttemptを作る。
6. Retryはnew Attempt。
7. late/stale operationはfail-closed。
8. Service/Authorization/idempotencyを必ず通す。

## 3. External LLM YAML

```yaml
jobs:
  classify:
    executor: external_llm
    with:
      text: ${{ inputs.text }}
    external:
      lease-minutes: 120
      on-lease-expiry: requeue
```

優先順位:

```text
Job external > Workflow settings > System default
```

System default: lease 60分、expiry `requeue`。

## 4. External activation

ready時1transaction:

1. final Input確定
2. continue-on-error snapshot
3. new Attempt
4. External Task作成
5. Job -> `waiting_external`
6. Event

one task / Attempt。

## 5. Task / Lease状態

Task:

```text
available | leased | completed | cancelled
```

Lease:

```text
active | expired | released | invalidated
```

Job statusはclaim中も `waiting_external`。

## 6. `task_claim`

Atomic transactionでcandidateを1件選びLease発行。

### 6.1 candidate selection order

External taskの選択順はinternal Job schedulingと同じ軸を使う。

1. Workflow Run priority DESC
2. Job priority DESC
3. Dynamic `order_rank` ASC
4. `source_order` ASC
5. `ready_at` ASC
6. Job Run ID deterministic tie-break

さらに以下を満たすもののみ:

- Task `available`
- Workflow Run pause/cancel無し
- Attempt non-terminal
- active lease無し
- Authorizationでclaim許可

これにより `claim_next` でも通常 `task_claim` でも順序を統一する。

### 6.2 lease

```text
lease_id
external_task_id
attempt_id
claimed_by
claimed_at
expires_at
released_at nullable
status
```

同一Task active leaseは最大1。

## 7. Lease expiry

### `requeue`

- Lease -> expired
- Task -> available
- same Attempt維持
- new Attemptを作らない

### `fail`

- Lease -> expired
- Task terminal
- Attempt -> failure `external_lease_expired`
- Retry policyへ

Lease expiryはWall Clockで進み、Workflow Pauseで停止しない。

## 8. `task_submit`

request:

```text
task_id
lease_id
result
artifacts optional
claim_next optional
request_id optional
```

受理条件:

- current task/Attempt
- current active unexpired lease
- Workflow cancel無し
- result JSON/schema/size validation
- `success_if`
- artifact metadata validation

1transactionで:

- Artifact metadata登録
- Attempt/Job terminal
- Lease release
- Task complete
- Event
- idempotency result

## 9. `claim_next`

submit transaction成功後、別transactionで通常 `task_claim` と同じselection/order/authorizationを使う。

next claim failureはsubmit成功をrollbackしない。

## 10. Pause / Cancel

Pause:

- new claim禁止
- existing Lease submit受理
- expiry clock継続

Cancel:

- available Task -> cancelled
- active Lease -> invalidated
- Attempt/Job -> cancelled
- late submit拒否

## 11. External Retry / Recovery

Retry:

- new Attempt + new Task
- old Task/Lease/Attempt保持

`requeue` expiryはRetryではない。

Recovery:

- active unexpired Lease維持
- expiredへpolicy適用
- available Task維持
- terminal/cancelled AttemptへLease発行禁止

## 12. Human YAML

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs.summary }}
```

Human Jobは `action/uses/runs-on/success_if/external/timeout-minutes/secrets` 禁止。

## 13. Human activation

ready時1transaction:

1. review Input snapshot
2. continue-on-error snapshot
3. new Attempt
4. Human Review `pending`
5. Job -> `waiting_review`
6. Event

one review / Attempt。

## 14. Human Review model

```text
review_id
job_run_id
attempt_id
status = pending | completed | cancelled
outcome nullable = approve | reject
comment nullable
actor_context nullable
created_at
completed_at nullable
```

Human lease / review timeoutはMVPなし。

## 15. `review_submit`

Atomic first-wins。

Approve -> Job success。

Reject -> Job failure `human_rejected`, retryable=false default。

failed Jobを人間操作で直接successへ変更するAPIはない。

## 16. Human Pause/Cancel/Retry

Pause: pending review維持、submit受理。

Cancel: review -> cancelled、late submit拒否。

Reject後Retry: new Attempt + new pending review。

## 17. Idempotency / Authorization

claim/submit/review read/writeはAuthorizationProviderを通す。

`task_claim`, `task_submit`, `review_submit` は optional `request_id`。

Idempotency scope/hash/TTLは `08` / `11` の共通規則に従う。

## 18. Events

```text
external_task_created
external_task_claimed
external_lease_expired
external_task_submitted
external_submit_rejected
external_task_cancelled
human_review_requested
human_review_submitted
human_review_cancelled
```

## 19. 受入条件

1. external/human timeout拒否
2. External activation one task/Attempt
3. task_claim scheduling order
4. concurrent claim exactly one
5. lease requeue/fail
6. stale/cancel submit拒否
7. claim_next同一selection rule
8. Human first-wins
9. Human timeout無し
10. retry/recovery/idempotency/authorization
