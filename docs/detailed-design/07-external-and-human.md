# 07. External / Human Executor 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `11`, `12`

## 1. 共通原則

1. External/Human JobはRunnerを占有しない。
2. `action/uses/runs-on/timeout-minutes`は禁止。
3. Secret参照は禁止。
4. Job Inputはactivation時snapshot。
5. 初回activation時にAttempt作成。
6. Retryはnew AttemptだがRetry request/schedule時にはまだ作らない。
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
Job external value > Workflow Run effective settings
```

Workflow Run effective settingsは`01`どおりRun System baseline + Workflow settingsから既にsnapshot済み。

Canonical default:

```text
lease-minutes=60
on-lease-expiry=requeue
```

Effective lease/expiryはAttempt/Task activation時にTask rowへsnapshotし、requeueでも変えない。Retryでも同じWorkflow Run Definition/effective settings snapshotから解決する。

## 3. External activation

1 transaction:

1. `02` persistent Input snapshot
2. Validator ID/version確認
3. effective lease config
4. new Attempt
5. External Task `available_at=activation time`
6. Job=`waiting_external`
7. Attempt Execution Log
8. Event

One Task / Attempt。

## 4. Task / Lease

Task=`available|leased|completed|cancelled`。

Lease=`active|expired|released|invalidated`。

`task_claim` candidate順:

1. Workflow Run priority DESC
2. Job priority DESC
3. `COALESCE(order_rank, 0)` ASC
4. source/generated source order ASC
5. **Task `available_at` ASC**
6. **Job `job_key` ASC**
7. Job Run ID ASC
8. Task ID ASC

Static Jobは`source_order=0`でよく、同条件ではstableな`job_key`がopaque UUIDより先にtie-breakする。Dynamic Jobは`order_rank/source_order`が先に効くため`order_by`/source array orderを維持する。

Job `ready_at`はExternal Task claim順には使わない。External Job初回activation後はJob status=`waiting_external`であり、claim可能時刻のSource of Truthは`external_tasks.available_at`。

Job/Task IDはopaque UUID系IDで、同一DB状態内の最終tie-breakとしてのみ使う。Cross-run再現順序を意味しない。

Atomic exactly-one claim。

Claim時 `expires_at = claim_time + Task snapshot lease_seconds`。

MVPにLease heartbeat/renew/extend/transfer無し。

## 5. Lease expiry

Canonical expiry boundary:

```text
now >= expires_at
```

この時点でLeaseはsubmit不可。Maintenance Loopがまだstatusを`expired`へ書き換えていなくても、`task_submit` transactionの再確認でexpiredとしてreject/expiry処理対象にする。

`requeue`:

- same Attempt
- Lease -> expired
- Task -> available
- `available_at = expiry processing time`
- new Attempt無し

`fail`:

- Lease -> expired
- Task terminal
- Attempt failure `external_lease_expired`
- `10` Retry policy

Pause中もclock継続。Maintenance Loopがclient traffic無しでexpiry処理。Restart時overdue Leaseを先処理。

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

Validatorにはresult + persistent Job Inputだけを渡しSecret/Runtime Handle無し。

Validation/domain failureでもsubmitは当該Attemptについてterminalに受理する。Retry可能なら`10`がJobをretry waitingへscheduleする。同submit transaction内でnew Attempt/Taskは作らない。

`task_submit.artifacts` はExternal Reference Artifact metadataのみ。Core binary upload/fetch無し。

## 7. Atomic submit + `claim_next`

`claim_next=true` のresponseをidempotent replay可能にするため、submit terminal DB state + optional next Task claim + completed idempotency resultを**同一SQLite write transaction**で確定する。

Long validation/filesystem prepareはtransaction前。

正規flow:

1. preliminary Lease/Task read
2. validation + PayloadStore prepare
3. `BEGIN IMMEDIATE`
4. idempotency key/hash再確認
5. Task/Lease owner + `now < expires_at` を再確認
6. Artifact metadata
7. Attempt/Job terminal or Retry schedule
8. Lease release / Task completed
9. 必要なdownstream DB bookkeeping
10. `claim_next=true`なら同じAuthorization + §4 orderingでavailable Task検索
11. candidateあり -> 同transactionでLease/Task leased
12. candidate無し -> `next_task=null`
13. full submit responseをidempotency rowへ保存
14. commit

Step 5で`now >= expires_at`ならresultを採用せず`lease_expired` conflict。Expiry policy適用は同transactionまたはMaintenanceのidempotent pathで一度だけ行う。

Transaction未commitならsubmitも未commit。Prepared orphanはcleanup対象。

Replayは保存responseを返し追加claimしない。

## 8. Pause / Cancel / Recovery

Pause:

- 当該paused Runのnew claim禁止
- existing Lease submit受理（expiry前のみ）
- lease expiry継続
- `claim_next`は他の`status=running` Runのeligible Taskをclaim可能だがpaused/queued concurrency waiterのTaskはcandidate外

Cancel:

- available Task cancelled
- active Lease invalidated
- Attempt/Job cancelled
- late submit reject

Recovery:

- active `now < expires_at` Lease維持
- `now >= expires_at` -> snapshotted policy
- terminal Attemptへnew Lease不可

## 9. Human Review

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs }}
```

Humanは:

```text
action
validator
uses
runs-on
success_if
external
timeout-minutes
Secret reference
outputs.schema
```

を禁止する。

Activation:

- persistent Input snapshot
- new Attempt
- one pending Review
- Attempt Execution Log
- Job=`waiting_review`

Outcome=`approve|reject`。

Approve:

- Job/Attempt success
- canonical Job Output=`null`
- PayloadStoreへ`null`を通常Outputとして保存

Reject:

- failure `human_rejected`
- retryable=false default
- current Job Output無し

Human lease/timeout無し。Concurrent submit first-wins。

Review metadataは`wf_review_info`がSource of TruthでJob Outputへ混ぜない。

## 10. Human control boundary

許可state change=pending Reviewへのapprove/rejectのみ。

無し:

```text
failed/cancelled Job -> success override
manual Job skip
completed Review rewrite
```

Reject後は`wf_retry`でretry pending snapshotを作り、schedulerがnew Attempt + new Reviewを作る。

Pause中review submit受理。Cancel後late submit reject。

## 11. Common Execution Log

External/Humanもinternalと同じAttempt Execution Logを持つ。

External:

```text
task available
claim/lease
lease expiry/requeue
submit accepted/rejected
validation result
Attempt terminal
```

Human:

```text
review pending
approve/reject
cancel/retry
Attempt terminal
```

Input/Output全bodyをCoreがLogへ自動複製しない。Input/Output APIが本文Source of Truth。SecretGuard/Authorization共通。

## 12. Idempotency / Authorization

`task_claim/task_submit/review_submit` optional request_id。Scope/hash/TTL=`08/11`。

All public read/write Authorization。

## 13. 受入条件

1. External Job > Run effective lease hierarchy
2. Task config snapshot/requeue固定
3. External arbitrary JSON result/spill
4. External Validator valid/invalid/exception
5. Validator failure schedules Retry but no inline new Attempt
6. External Artifact reference only
7. concurrent claim exactly one
8. claim order uses task available_at, not job ready_at
9. stable job_key before opaque ID tie-break
10. claim/claim_next same ordering
11. lease boundary `now >= expires_at`
12. requeue updates available_at
13. submit + next claim + idempotency one transaction
14. submit crash/replay no double claim
15. no lease heartbeat/renew
16. lease requeue/fail/Pause/Recovery
17. stale/cancel submit reject
18. Human outputs.schema forbidden
19. Human approve Output exactly null
20. Human reject Output unavailable
21. Human first-wins/retry
22. Human override/skip/rewrite無し
23. common Execution Log
24. idempotency/authorization
