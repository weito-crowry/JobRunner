# 13. Testing 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`〜`12`

## 1. 基本原則

1. 状態遷移はintegration test必須。
2. SQLite concurrency/transactionは実DB。
3. Runner/Action Runnerは実Process E2E。
4. External/Human/Retry/Recoveryはnegative case必須。
5. Definition/Expressionはtable-driven。
6. 時刻依存はClock abstraction。
7. 競合testはbarrier/hookを使う。

## 2. Test layers

```text
Unit
Integration
Process Integration
Adapter Contract
End-to-End
Recovery / Failure Injection
```

## 3. Definition / Expression

必須negative:

- duplicate YAML key / merge key / custom tag
- unknown key
- bad Job ID / missing needs / DAG cycle
- Dynamic `foreach.parent`を含むcycle
- executor field conflict
- envでSecret参照
- external/human/reusableでSecret参照
- external/human/reusable `timeout-minutes`
- invalid CEL/JMESPath
- missing/null/type mismatch

Condition helper:

- `success/failure/cancelled/always`
- continue-on-error failureはeffective success
- **通常Job skippedはeffective successではなく後段default skip**
- Dynamic group内skippedはgroup集約規則を通す
- explicit if false
- Nested Dynamic parent + declared needs

## 4. Input / Output / Secret

- Workflow required/default/extra/type
- `$base` shallow override
- Retry Input固定
- Secret reference固定 + Attemptごとrematerialize
- Secret非永続化
- Output schema
- NaN/Infinity
- 4MiB default limit/override

### SecretGuard regression

current Attemptへ注入したSecret値について:

- Outputのstring全体一致 -> reject
- Output文字列の部分一致 -> reject
- nested object/list内 -> reject
- `state.set` -> reject
- Artifact URI/metadata -> reject
- Event/error details -> reject
- stdout/stderr/log -> `[REDACTED]`
- Base64等に変形された値は保証対象外

## 5. Scheduling

- Workflow priority
- Job priority
- Dynamic order/source
- pool routing
- one internal running / Workflow Run
- multiple Workflow Runs parallel
- pause/resume
- priority update non-preemptive

Internal claim競合はexactly one。

## 6. Runner / IPC

- start/ready/log/progress/step/result/error
- malformed JSON/protocol mismatch
- stdout/stderr capture
- Action exception/crash/hang
- heartbeat継続
- main-loop stall detection
- old runner fencing
- restart suppression
- Parent shutdownとWorkflow cancelの区別
- Parent restartでold running Attempt -> runner_lost Recovery

## 7. Timeout

- internal no-timeout long action success
- internal timeout -> cancel -> child terminate -> `job_timeout`
- timeout後Retry
- external/human/reusable timeout definition reject

## 8. Dynamic Jobs

- 0/1/N
- stable key/index fallback
- parent別同raw key非衝突
- percent encoding
- full logical job_keyに固定長上限無し
- 1000 allowed / 1001 rollback
- nested 2/3段以上
- arbitrary-depth representative
- parent edge cycle
- parent+needs helper
- order_by type/order
- atomic expansion crash/restart
- group status running/completed
- group conclusion success/failure/blocked/cancelled
- full-key Output/Artifact lookup

## 9. Reusable Workflow

- registered ID
- relative pathはcaller source directory基準
- root traversal/symlink escape
- non-filesystem caller relative reject
- binding固定
- source変更後Retry same binding
- parent-child output/state isolation
- direct/indirect cycle
- direct Child control reject
- Dynamic + Reusable
- restart duplicate防止

## 10. External LLM

- activation one Attempt/Task
- concurrent claim exactly one
- candidate ordering = Workflow priority -> Job priority -> dynamic order -> source -> ready -> id
- claim_next同一selection
- lease expiry requeue same Attempt
- lease expiry fail -> Retry
- stale/cancel submit reject
- Pause中existing submit可/expiry clock継続
- restart valid/expired lease

## 11. Human

- pending review
- approve/reject
- concurrent first-wins
- cancel/late submit
- pause中submit
- Retry new review
- timeout無し

## 12. Retry / Recovery

- auto retry disabled default
- max-attempts/condition/backoff
- manual retry completed/failure Run reopen
- run_attempt増加
- success/cancelled Run retry reject
- blocked descendants再評価
- terminal RunはRecoveryだけでreopenしない
- retry backoff restart
- Dynamic/Child duplicate無し

## 13. Persistence

- fresh/incremental migration
- WAL/FK/busy timeout
- internal running partial unique
- Dynamic expansion unique
- no logical job_key length CHECK
- Reusable uniqueness
- External active lease unique
- Human one review/Attempt
- state current+history atomic
- concurrency race

## 14. Idempotency

- same Actor/AccessScope + same key/hash -> replay
- same scope + same key/different hash -> conflict
- different Actorはsame keyでも独立
- different AccessScopeはsame keyでも独立
- concurrent same key -> single side effect
- TTL expiry後、expired row残存状態でsame keyを新requestとして再利用
- `task_claim` replayは新Lease/別Taskを追加しない

## 15. Authorization / Security

- AllowAll/Deny
- list/read/write全public operation authorize
- filtered AccessScope
- log path traversal reject
- Artifact URI auto-fetch無し
- arbitrary shell非搭載

## 16. Platform / CI

Windows 11 / Linux。

推奨:

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform-matrix
```

## 17. MVP completion gate

1. `01`〜`12`受入条件対応test
2. migration pass
3. Windows/Linux process integration
4. concurrent claim
5. restart recovery
6. External/Human E2E
7. Dynamic 1000 + nested + rollback
8. Reusable nested/cycle/binding
9. SecretGuard/non-persistence
10. idempotency scope/TTL
11. adapter contract

WebUI E2EはWebUI詳細設計後。
