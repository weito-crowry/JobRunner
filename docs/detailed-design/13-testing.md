# 13. Testing 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`〜`12`

## 1. 基本原則

1. 状態遷移はintegration testを必須にする。
2. SQLite concurrency/transactionは実DBで検証。
3. Runner/Action Runnerは実Process E2Eを持つ。
4. External/Human/Retry/Recoveryはnegative case必須。
5. Definition/Expressionはtable-driven。
6. 時刻依存はClock abstraction。
7. 競合testはbarrier/hookを使いrandom sleepへ依存しない。

## 2. Test layers

```text
Unit
Integration
Process Integration
Adapter Contract
End-to-End
Recovery/Failure Injection
```

## 3. Definition / Expression

必須negative:

- duplicate YAML key / merge key / custom tag
- unknown key
- bad Job ID / missing needs / DAG cycle
- **Dynamic foreach.parentを含むcycle**
- executor field conflict
- envでSecret参照
- external/human/reusableでSecret参照
- external/human/reusable `timeout-minutes`
- invalid CEL/JMESPath
- missing/null/type mismatch

Condition helper:

- success/failure/cancelled/always
- continue-on-error effective success
- skipped dependency
- explicit if false
- **Nested Dynamic parent + declared needsのcondition dependency set**

## 4. Input / Output / Secret

- Workflow required/default/extra/type
- `$base` shallow override
- Retry Input固定
- Secret reference固定 + Attemptごとrematerialize
- Secret非永続化
- Output schema
- NaN/Infinity
- 4MiB default limit / override

## 5. Scheduling

- Workflow priority
- Job priority
- Dynamic order_rank/source_order
- pool routing
- one internal running per Workflow Run
- multiple Workflow Runs parallel
- pause/resume
- priority update non-preemptive

Internal claim競合はexactly one。

## 6. Runner / IPC

- start/ready/log/progress/step/result/error
- malformed JSON / protocol mismatch
- stdout/stderr capture
- action exception/crash/hang
- heartbeat継続
- main-loop stall detection
- old runner fencing
- restart suppression

## 7. Timeout

- internal no-timeout long action success
- internal timeout -> cancel -> child terminate -> `job_timeout`
- timeout後Retry
- **external/human/reusable timeout definition reject**

## 8. Dynamic Jobs

必須:

- 0/1/N
- stable key/index fallback
- parent別同raw key非衝突
- special char percent encoding
- **full logical job_keyに固定長上限無し**
- 1000 allowed / 1001 rollback
- nested 2/3段以上
- arbitrary-depth representative
- parent edge cycle
- parent+needs helper
- order_by type/order
- atomic expansion crash/restart
- group status `running/completed`
- group conclusion success/failure/blocked/cancelled
- full-key Output/Artifact lookup

## 9. Reusable Workflow

- registered ID
- relative pathはcaller source directory基準
- root traversal/symlink escape
- non-filesystem caller relative reject
- binding固定
- source変更後Retryでsame binding
- parent-child output/state isolation
- direct/indirect cycle
- direct child control reject
- Dynamic + Reusable
- restart duplicate防止

## 10. External LLM

- activation one Attempt/Task
- concurrent claim exactly one
- **candidate ordering = Workflow priority -> Job priority -> dynamic order -> source -> ready -> id**
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
- running internal runner_lost
- retry backoff restart
- Dynamic/Child duplicate無し

## 13. Persistence

- fresh/incremental migration
- WAL/FK/busy timeout
- internal running partial unique
- Dynamic expansion partial unique
- Reusable uniqueness
- External active lease unique
- Human one review/Attempt
- state current+history atomic
- concurrency race
- no fixed DB length check for logical job_key

## 14. Idempotency

対象operationごとに:

- same Actor/AccessScope + same key/hash -> replay
- same scope + same key/different hash -> conflict
- **different Actorはsame keyでも独立**
- **different AccessScopeはsame keyでも独立**
- concurrent same key -> single side effect
- TTL expiry後、expired rowがDBに残ったままsame keyを新requestとして再利用

## 15. Authorization / Security

- AllowAll/Deny
- list/read/write全public operationがauthorize通過
- filtered AccessScope
- log path traversal reject
- Artifact URI auto-fetch無し
- arbitrary shell非搭載

## 16. Platform / CI

少なくともWindows 11 / Linux。

CI推奨:

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform-matrix
```

1000 Dynamic Jobs等の重いstressはPR/nightly分離可。

## 17. MVP completion gate

1. `01`〜`12`の受入条件に対応test
2. migration pass
3. Windows/Linux process integration
4. concurrent claim
5. restart recovery
6. External/Human E2E
7. Dynamic 1000 + nested + atomic rollback
8. Reusable nested/cycle/binding
9. Secret non-persistence
10. idempotency scope/TTL
11. adapter contract

WebUI E2EはWebUI詳細設計後に追加する。
