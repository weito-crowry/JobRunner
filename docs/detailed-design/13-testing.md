# 13. Testing 詳細設計

- Status: Draft v0.6
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

Python 3.10で以下のimportと代表処理をCI確認する。

```text
ruamel.yaml >=0.19,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

## 3. Definition / Expression

- duplicate/merge/custom tag reject
- unknown key
- `env` literal-only、expression/Secret reject
- bad needs / Dynamic parent cycle
- executor field conflict
- external/human/reusable timeout reject
- invalid CEL/JMESPath
- null/missing/type strictness
- normal skipped dependency -> downstream default skip
- continue-on-error failure -> effective success
- Dynamic group aggregate / Nested parent helper

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
- downstream `needs.*.outputs` がinline/blobで同一value
- Workflow Output spill
- blob temp write -> atomic rename -> DB commit
- DB commit failure orphan blob cleanup
- referenced blob missing -> `payload_missing`
- digest mismatch -> `payload_digest_mismatch`
- Attempt historyのold Output維持

## 5. SecretGuard

Current Attempt known Secret:

- Output exact/substring/nested -> reject
- large Output spill前reject
- state.set -> reject
- Artifact metadata/URI -> reject
- Event/error -> reject
- log -> redact
- managed Artifact text/binary byte match -> reject
- chunk境界match -> reject
- transformed Secretは保証外

## 6. Scheduling / Claim

- Workflow priority
- Job priority
- Dynamic order/source
- pool routing
- one internal running / Workflow Run
- multiple Workflow Runs parallel
- pause/resume
- non-preemptive priority update
- internal claim exactly-one race
- External claimは同じordering軸

## 7. Runner / IPC

- start/ready/log/progress/step/error/exiting
- Runtime Handle request_id correlation
- state_get/set response
- Artifact put/materialize response
- Runtime request待ち中cancel
- Action result file protocol
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
- 1000 allowed / 1001 rollback
- nested 2/3+ depth
- arbitrary-depth representative
- parent edge cycle
- parent+needs helper
- order type/order
- expansion crash/restart
- group status/conclusion
- arbitrary JSON Output aggregation
- full-key Artifact lookup

## 10. Reusable Workflow

- registered ID
- relative pathはcaller source directory基準
- traversal/symlink escape
- non-filesystem caller relative reject
- binding固定 / Retry same binding
- Parent/Child Output/state isolation
- cycle
- direct Child control reject
- Dynamic + Reusable
- restart duplicate防止
- Child Workflow Output large spill

## 11. Managed ArtifactStore

- put_file durable copy
- source work_dir traversal reject
- immutable same-name generations
- managed materialize current work_dir
- materialize destination safety
- Retry current generation
- managed retention deletes data
- store failure -> no metadata/current exposure

External Artifact:

- URI metadata register
- no fetch
- Core retention does not delete external data
- External LLM `task_submit.artifacts`はexternal reference only

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

## 14. Automatic / Manual Retry

- auto retry default off
- max-attempts / condition / backoff
- Retry target Input fixed
- manual retry completed/failure Run reopen
- `run_attempt++`
- success/cancelled Run retry reject
- blocked/skipped descendants re-evaluate
- failed non-target descendant not auto-retried
- terminal Run not reopened by Recovery

## 15. Same-Run Result Reuse

### Scope

- same Workflow Runだけreuse可能
- 別Workflow Runの同一Job/同一Inputでもautomatic reuseしない

### Reuse key positive

同一Runで以下が同一ならmatchすること:

- final persistent Job Input digest
- direct upstream Artifact identity (`artifact_id` + digest if present)
- entire Workflow Definition hash
- executor identity
  - internal `action_id + version`
  - external protocol identity
  - human protocol identity
  - reusable binding Child Definition/action versions

### Negative / ineligible

- Input changed -> mismatch
- upstream Artifact generation changed -> mismatch
- Definition hash changed -> mismatch
- Action version changed/unavailable -> mismatch/fail-closed
- spilled Output blob missing/digest bad -> cannot reuse
- ActionがRuntime Handle `state.get`使用 -> `reuse_eligible=false`
- Actionがfrozen Input/direct upstream以外のArtifactをdynamic materialize -> false

### Manual Retry descendant

- successful descendant + key match -> existing success維持 + `job_result_reused`
- mismatch/ineligible/Payload missing -> `successful_job_result_not_reusable`
- mismatch時にsame Job Runへnew InputのAttemptを自動作成しない
- new Workflow Runが必要

### Retention

Reuse候補のPayload/managed Artifactがretentionで欠落した場合、silent reuseせずfail-closed。

## 16. Recovery

- Parent restart old running -> `runner_lost`
- queued/waiting/backoff restore
- Dynamic expansion duplicate無し
- Child duplicate無し
- `reuse_check_pending` restore
- completed RunはRecoveryだけでreopenしない

## 17. Persistence / Idempotency

- migration/WAL/FK/busy timeout
- output inline/blob column constraints
- `reuse_context_json/reuse_key/reuse_eligible`
- `reuse_check_pending`
- internal running unique
- Dynamic/Reusable/External/Human unique
- state current/history atomic
- concurrency race
- idempotency Actor/AccessScope isolation
- TTL replay/conflict
- TTL expired row replacement
- task_claim idempotent replay no extra Lease

## 18. Authorization / Security

- AllowAll/Deny/filtered scope
- all public read/write authorize
- log path safety
- Artifact URI no auto-fetch
- arbitrary shell無し

## 19. Platform / CI

Windows 11 / Linux。

```text
lint/typecheck
unit
integration-sqlite
process-integration
adapter-contract
recovery
platform-matrix
```

## 20. MVP completion gate

1. `01`〜`12`受入条件対応
2. dependency import Python3.10+
3. migration
4. process integration Windows/Linux
5. claim/concurrency races
6. PayloadStore inline/spill/crash
7. Managed ArtifactStore
8. External/Human E2E
9. Dynamic1000/nested/rollback
10. Reusable binding/cycle
11. SecretGuard
12. same-run Result Reuse positive/negative
13. idempotency scope/TTL
14. adapter contract

WebUI E2Eは後続。
