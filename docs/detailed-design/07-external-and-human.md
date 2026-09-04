# 07. External / Human Executor 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`, `10-retry-recovery-cancel.md`

## 1. 目的

`external_llm` と `human` executorのAttempt生成、claim/lease/submit、review、Cancel、Retry、Recoveryを定義する。

## 2. 共通原則

1. External/Human JobはRunnerを占有しない。
2. `action/uses/runs-on`は禁止。payloadは`with`から作る。
3. `${{ secrets.* }}`はExternal/Human `with`では禁止。
4. Job Inputはactivation時にsnapshot。
5. Job activation時にAttemptを作る。
6. Retryは新Attempt。
7. stale/late operationをfail-closedで拒否。
8. Service/Authorization/idempotency境界を必ず通す。

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

設定優先順位:

```text
Job external > Workflow settings > System default
```

System default:

```text
lease = 60 minutes
on expiry = requeue
```

## 4. External activation

ready時に1transactionで:

1. final Input確定
2. continue-on-error snapshot
3. new Attempt作成
4. External Task作成
5. Job -> `waiting_external`
6. Event

同じAttemptにExternal Taskは1件だけ。

## 5. External Job/Task状態

Job statusはclaim中も`waiting_external`。

Task status:

```text
available
leased
completed
cancelled
```

Lease status:

```text
active
expired
released
invalidated
```

OwnershipはJob statusではなくTask/Leaseで管理する。

## 6. `task_claim`

Atomic transaction:

1. Authorization
2. candidate task選択
3. Workflow Run pause/cancel/Attempt terminalを再確認
4. active lease無しを確認
5. Lease作成
6. Task -> leased
7. Event/idempotency結果
8. payload返却

同一taskの同時claimはexactly one success。

Lease:

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

## 7. Lease expiry

### `requeue`

- current Lease -> expired
- Task -> available
- **同じAttemptを維持**
- 新Attemptを作らない
- 次claimで新Leaseを発行

Ownership windowが切れただけでexecution failureとは数えない。

### `fail`

- Lease -> expired
- Task terminal
- Attempt -> failure `external_lease_expired`
- Retry policyへ渡す

Expired/released/invalidated Leaseからsubmit不可。

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

1. Authorization/idempotency
2. task/Attempt current
3. lease current/active/unexpired
4. Workflow cancel無し
5. result JSON-compatible + size/schema
6. `success_if`評価可能
7. artifact metadata妥当

`artifacts`はExternal executorが親システム側に既に保存した実体の参照metadata配列。Coreが実体upload/fetchしない。

### 8.1 atomic submit

1transactionで:

- result validation
- Artifact metadata登録
- Attempt/Job terminal transition
- Lease release
- Task complete
- Event
- idempotency result

Output/Artifact current世代解決は通常Jobと同じ。

## 9. `claim_next`

submit transaction成功後、同じService call内で別transactionの`task_claim`を試せる。

```json
{
  "submitted": true,
  "next_task": null
}
```

next claimが失敗/候補無しでもsubmit成功をrollbackしない。

次候補の選択は通常External scheduling/Authorization/priorityに従う。

## 10. `task_info`

read-onlyで:

```text
task/Job/Attempt status
input snapshot
current lease summary
claim history
submit summary
failure/output/artifact availability
```

を返せる。Secret valueは存在しない。

## 11. External Cancel

Workflow cancel:

- available Task -> cancelled
- active Lease -> invalidated
- Job/Attempt -> cancelled
- late submit拒否

外部clientへpush cancelは保証しない。

Pause:

- new claimは禁止
- existing active Lease submitは受理
- Lease expiry clockはPauseで停止しない

## 12. External Retry

Attempt failure後にRetryする場合:

- Jobをretry waiting/queuedへ
- retry execution開始時にnew Attempt + new External Task
- old Task/Lease/Attempt履歴保持

`requeue` lease expiryはRetryではないためnew Attemptを作らない。

## 13. External Recovery

Runtime再起動:

- active unexpired Lease維持
- expired Leaseへexpiry policy
- available task維持
- terminal/cancelled Attemptにnew Lease不可
- one task/Attempt uniqueにより重複作成防止

## 14. Human YAML

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs.summary }}
```

Human Jobに`action/uses/runs-on/success_if/external/secrets`は不可。

## 15. Human activation

ready時1transactionで:

1. final review Input snapshot
2. continue-on-error snapshot
3. new Attempt
4. Human Review row `pending`
5. Job -> `waiting_review`
6. Event

同一AttemptにHuman Review rowは1件だけ。

## 16. Human Review model

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

`outcome`はpending中null。

Human leaseは持たない。

## 17. `review_submit`

request:

```text
review_id or workflow_run_id+job_run_id+attempt_id
outcome approve|reject
comment optional
request_id optional
```

Atomic first-wins:

- `pending`のみupdate可能
- concurrent submitの1件だけ成功
- review + Attempt/Job terminal + Event + idempotency resultを1transaction

Approve:

```text
Job conclusion = success
```

Reject:

```text
Job conclusion = failure
failure.code = human_rejected
retryable = false default
```

failed Jobを直接successへ上書きするAPIは無い。

## 18. Human Cancel/Pause/Retry

Cancel:

- pending review -> cancelled
- Attempt/Job -> cancelled
- late submit拒否

Pause:

- pending reviewを保持
- review submitは受理

Reject後Retry:

- new Attempt + new pending Review
- old review履歴保持

## 19. Authorization / Actor

Claim/Submit/Review read/writeすべてAuthorizationProviderを通す。`claimed_by`やreview actorは親側ActorContext由来。不要な個人情報をCoreが追加収集しない。

## 20. Idempotency

`task_claim`はrequest_id optional。`task_submit/review_submit`もoptional request_idを受ける。

同じoperation/scope/request_id + 同一requestは最初のresponseを返す。内容が異なる場合`idempotency_conflict`。

Lease/task/reviewの副作用とidempotency resultは可能な限り同じDB transactionで確定する。

## 21. Events

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

## 22. Failure code

```text
external_lease_conflict
external_lease_expired
external_submit_invalid
external_result_invalid
external_task_cancelled
human_rejected
human_review_conflict
human_review_cancelled
```

## 23. 受入条件

1. External field constraints/Secret拒否
2. activation -> one Attempt/Task
3. concurrent claim exactly one
4. lease requeue同Attempt
5. lease fail -> Retry policy
6. stale/expired/cancel submit拒否
7. submit Output/Artifact atomic
8. claim_next submit非rollback
9. pause existing submit/lease clock
10. Retry -> new Attempt/Task
11. restart lease維持/expiry
12. Human activation pending row/outcome null
13. approve/reject/concurrent first-wins
14. Human cancel/retry/pause submit
15. idempotency/authorization
