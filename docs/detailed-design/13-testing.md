# 13. Testing 詳細設計

- Status: Draft v0.5
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
7. 競合testはbarrier/hook。

## 2. Foundation dependencies

Python3.10で:

```text
ruamel.yaml >=0.19,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

のimport / representative parse-validationをCI確認。

## 3. Definition / Expression

- duplicate/merge/custom tag reject
- env literal-only / expression/Secret reject
- bad needs / Dynamic parent cycle
- executor field conflict
- external/human/reusable timeout reject
- invalid CEL/JMESPath
- null/missing/type strictness
- normal skipped dependency -> downstream default skip
- continue-on-error effective success
- Dynamic group aggregate + Nested parent helper

## 4. JSON Output / PayloadStore

Action/External result:

```text
null
boolean
number
string
array
object
```

すべてpositive test。

Optional JSON Schemaを各shapeで検証。

Transparent storage:

- threshold-1 -> inline
- threshold -> inline
- threshold+1 -> blob
- multi-MiB JSON -> success
- downstream `needs.*.outputs` がinline/blob同じvalue
- Workflow Output spill
- blob temp write -> atomic rename -> DB commit
- DB commit failure orphan blob cleanup
- referenced blob missing -> `payload_missing`
- digest mismatch -> `payload_digest_mismatch`
- Attempt historyのold Output維持

## 5. SecretGuard

Current Attempt known Secretについて:

- Output exact/substring/nested -> reject
- large Output spill前reject
- state.set -> reject
- Artifact metadata URI -> reject
- Event/error -> reject
- log -> redact
- managed Artifact text/binary byte match -> reject
- chunk境界match -> reject
- transformed Secretは保証外

## 6. Scheduling

- Workflow/Job/Dynamic priority order
- pool routing
- one internal running / Workflow Run
- multiple Runs parallel
- pause/resume
- non-preemptive priority update
- internal claim exactly-one race

## 7. Runner / IPC

- start/ready/log/progress/step/error/exiting
- Runtime Handle request_id correlation
- state_get/set response
- artifact put/materialize response
- request待ち中cancel
- Action return result file protocol
- giant JSON resultでもIPC frame小さい
- result path traversal reject
- size/digest mismatch
- child crash/hang
- heartbeat/main-loop stall
- Parent shutdown vs Workflow cancel
- old Runner fencing/restart suppression

## 8. Timeout

- internal no-timeout long Action
- internal timeout -> cancel/terminate -> retry
- external/human/reusable timeout reject

## 9. Dynamic Jobs

- 0/1/N
- stable/index key
- parent別same raw key
- percent encoding
- fixed job_key length limit無し
- 1000/1001 rollback
- nested 2/3+ depth
- parent cycle
- parent+needs helper
- order type/order
- expansion crash/restart
- group status/conclusion
- arbitrary JSON Output aggregation

## 10. Reusable Workflow

- registered ID
- caller-directory relative resolution
- traversal/symlink escape
- non-filesystem relative reject
- binding固定 / retry same binding
- parent-child output/state isolation
- cycle
- direct Child control reject
- Dynamic+Reusable
- restart duplicate防止

## 11. Managed ArtifactStore

- put_file copies to durable store
- put source work_dir traversal reject
- immutable same-name generations
- managed Artifact materialize current work_dir
- materialize destination safety
- retry current generation resolution
- managed retention deletes data
- store failure -> no metadata/current pointer

External Artifact:

- register URI metadata
- no fetch
- Core retention does not delete external data
- External LLM task_submit Artifactはexternal reference only

## 12. External LLM

- arbitrary JSON result
- large result spill
- activation one Task/Attempt
- candidate ordering / concurrent claim
- claim_next same selection
- lease requeue/fail
- stale/cancel submit reject
- Pause/recovery

## 13. Human

- pending/approve/reject
- Job Output null
- concurrent first-wins
- cancel/pause/retry
- timeout無し

## 14. Retry / Recovery

- auto retry default off
- max/condition/backoff
- manual retry completed/failure reopen
- run_attempt++
- success/cancelled retry reject
- blocked/skipped descendants reevaluate
- terminal Run not reopened by recovery
- Parent restart old running -> runner_lost
- Dynamic/Child duplicate無し

## 15. Persistence / Idempotency

- migration/WAL/FK/busy timeout
- output inline/blob column constraints
- internal running unique
- Dynamic/Reusable/External/Human unique
- state current/history atomic
- concurrency race
- idempotency actor/scope isolation
- TTL replay/conflict
- TTL expired row replacement
- task_claim idempotent replay no extra Lease

## 16. Authorization

- AllowAll/Deny/filtered scope
- all public read/write authorize
- log path safety
- Artifact URI no auto-fetch
- arbitrary shell無し

## 17. Platform / CI

Windows11 / Linux。

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform-matrix
```

## 18. MVP completion gate

1. `01`〜`12`受入条件対応
2. dependency imports Python3.10+
3. migration
4. process integration Windows/Linux
5. claim/concurrency races
6. PayloadStore inline/spill/crash
7. Managed ArtifactStore
8. External/Human E2E
9. Dynamic1000/nested/rollback
10. Reusable binding/cycle
11. SecretGuard
12. idempotency scope/TTL
13. adapter contract

WebUI E2Eは後続。
