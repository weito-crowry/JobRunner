# 07. External / Human Executor 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/01-workflow-definition.md`
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/03-runtime-and-scheduling.md`
  - `docs/detailed-design/10-retry-recovery-cancel.md`

## 1. 目的

本書は JobRunner における `external_llm` executor と `human` executor の状態遷移、claim / lease、submit、review、Cancel、Retry、Recovery を定義する。

## 2. 共通原則

1. External / Human Job は Runner を占有しない。
2. Job Input は ready 時点で固定する。
3. Retry は新しい Attempt を作る。
4. late submit / stale lease は受理しない。
5. 状態変更は Service layer を経由する。
6. state-changing operation は idempotency key を受け付けられる。

## 3. External LLM executor

YAML:

```yaml
jobs:
  classify:
    executor: external_llm
    with:
      text: ${{ inputs.text }}
```

`external_llm` Job は `action` / `uses` / `runs-on` を持たない。

## 4. External Job 状態

概念遷移:

```text
queued
  ↓ ready
waiting_external
  ↓ task_claim
claimed / leased
  ↓ task_submit
completed
```

DB表現では Job status と lease 状態を分けてもよい。

Job 自体は claim 中も `waiting_external` とし、ownership は external_tasks / external_leases で管理する設計を基本とする。

## 5. task_claim

`task_claim` は atomic に以下を行う。

1. claim可能な external task を選択
2. current lease が存在しない / expired であることを確認
3. lease を作成
4. task と lease を結び付ける
5. Eventを記録
6. task payload を返す

複数 caller が同時に同一 task を claim した場合、1 caller だけ成功する。

## 6. Lease

最低限以下を持つ。

```text
lease_id
task_id
workflow_run_id
job_run_id
attempt_id
claimed_by
claimed_at
expires_at
released_at
status
```

MVP default duration:

```text
60 minutes
```

System -> Workflow -> Job の順で override 可能とする。

## 7. Lease expiry

期限切れ時の policy:

```text
requeue
fail
```

既定は `requeue` を推奨するが、最終値は設定可能にする。

`requeue` 時は同じ Attempt の ownership を解除して再claim可能にする方式ではなく、External executor上の同一 Attempt を再claim可能にする。

一方、execution failure として数えたい親システムは `fail` を選択できる。

## 8. task_submit

requestには最低限以下を含める。

```text
task_id
lease_id
result
request_id optional
claim_next optional
```

受理条件:

1. task が current
2. lease_id が current lease と一致
3. lease が未失効
4. Job / Attempt が submit可能状態
5. Workflow Run が cancel 済みでない
6. result が protocol / schema validation を通る

不一致は fail-closed で拒否する。

## 9. submit result

External result は通常 Job Output と同様に扱う。

- JSON-compatible
- output schema optional
- success_if optional
- artifact references optional

validation failure は Job failure とする。

## 10. submit chaining

`claim_next: true` を指定した場合、current submit 成功transaction確定後に次の claim 候補を探せる。

返却例:

```json
{
  "submitted": true,
  "next_task": {"task_id": "...", "lease_id": "...", "input": {}}
}
```

次候補が無ければ `next_task: null`。

submit 自体の成功と next claim の失敗を混同しない。

## 11. task_info

read-only。

返却候補:

```text
task status
job / attempt
current lease
lease expires_at
claim history
submit history summary
input snapshot
output availability
failure
```

## 12. External cancel

Workflow Run / Job cancel 時:

- 未claim task: cancelled
- claim済み task: current lease を invalidated
- 以後の submit を拒否

外部 client への push cancel は保証しない。

## 13. External Retry

Job Retry では新しい Attempt と新しい external task を作る。

旧 task / lease は履歴として保持する。

## 14. External Recovery

Runtime 再起動時:

- valid lease は expires_at まで維持
- expired lease は policy 適用
- cancelled / terminal Attempt に新leaseを発行しない
- duplicate task生成を防ぐ

## 15. Human executor

YAML:

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs.summary }}
```

`human` Job は Runner を使わない。

## 16. Human Job 状態

```text
queued
  ↓ ready
waiting_review
  ↓ review_submit
completed
```

MVP outcome:

```text
approve
reject
```

## 17. Review payload

review request側で人間へ見せる data は Job Input / upstream output / artifact refs から作る。

review submit:

```text
job_run_id
attempt_id
outcome
comment optional
request_id optional
```

## 18. Approve / Reject

`approve`:

- Job conclusion = success
- Job Output を定義している場合、review metadata を output として公開可能

`reject`:

- Job conclusion = failure
- failure code = `human_rejected`

人間操作で任意 failed Job を success に直接書き換えるAPIは提供しない。

## 19. Human lease

MVPでは Human Review に lease を持たせない。

複数 reviewer が同時submitした場合、最初に terminal transition を確定した request のみ成功し、後続は conflict を返す。

## 20. Human Retry

Reject後にRetryした場合、新Attemptを作り再度 `waiting_review` へ進む。

元review履歴は残す。

## 21. Human Cancel

waiting_review 中のJobがcancelされたら cancelled。

その後の review_submit は拒否する。

## 22. Authorization

claim / submit / review_submit は AuthorizationProvider を通す。

ActorContextをEventへ記録する。

`claimed_by` は親システムが渡すactor/client identifierでよい。

## 23. Event

代表:

```text
external_task_created
external_task_claimed
external_lease_expired
external_task_submitted
external_submit_rejected
human_review_requested
human_review_submitted
human_review_rejected
```

## 24. Failure code

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

## 25. 受入条件

1. external task claim atomicity
2. duplicate claim conflict
3. valid submit
4. stale lease submit拒否
5. expired lease requeue/fail
6. claim_next
7. cancel後late submit拒否
8. retryで新task
9. runtime restart後lease維持
10. human approve
11. human reject
12. duplicate review conflict
13. human cancel
14. authorization hook通過
15. idempotent submit/review
