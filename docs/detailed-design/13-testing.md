# 13. Testing 詳細設計

- Status: Draft v0.9
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

Python 3.10で:

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

のinstall/import/代表処理をCI確認する。

追加確認:

- CEL custom function bindingで`jmespath(...)` helperを登録・実行できる
- Windows Python3.10でcel-python/google-re2依存をinstallできる
- dependency license inventoryが設計記載と一致する

## 3. Definition / Expression

- duplicate/merge/custom tag reject
- unknown key
- Input `nullable=false/true`
- required+nullable
- `default:null` nullable true/false
- extra Input reject
- env literal-only / expression/Secret reject
- bad needs / Dynamic parent cycle
- executor field conflict
- Action version empty/non-string reject
- concurrency group non-string/null/empty reject
- concurrency group case-sensitive identity
- priority signed64 min/max/overflow
- NaN/Infinity numeric field reject
- external/human/reusable timeout reject
- invalid CEL/JMESPath
- null/missing/type strictness
- normal skipped dependency -> downstream default skip
- continue-on-error failure -> effective success
- Dynamic group aggregate / Nested parent helper

## 4. JSON Output / PayloadStore

Job result:

```text
null
boolean
number
string
array
object
```

すべてpositive。

Workflow Outputはname mappingによるobjectになることを確認。

Optional JSON Schemaを各shapeで検証。

Transparent storage:

- threshold-1 -> inline
- threshold -> inline
- threshold+1 -> blob
- multi-MiB JSON -> success
- downstream `needs.*.outputs` がinline/blob同一value
- Workflow Output spill
- blob temp write -> atomic rename -> DB commit
- DB commit failure orphan blob cleanup
- referenced blob missing -> `payload_missing`
- digest mismatch -> `payload_digest_mismatch`
- Attempt history old Output維持

## 5. Service / MCP / HTTP Adapter

Canonical names:

- `wf_definition_list/info`
- `wf_run_list/info`
- ambiguous `wf_list/wf_info` alias無し
- namespaced MCP names

Output:

- `wf_run_info`はOutput metadataのみ、本文無し
- `wf_output_info`はinline/blobで同じmetadata contract
- `wf_output_read`はinline/blobで同じJSON value
- Job current Output / specific Attempt Output / Workflow Outputのsource resolution
- optional JMESPath `select`
- invalid select -> structured error
- MCP large full response -> `response_too_large`, silent truncate無し
- selectで小さいresponseなら成功
- Output read authorization / AccessScope

HTTP v1:

- 全canonical route/methodが`11`と一致
- path ID percent-encoding
- list/query mapping
- `Idempotency-Key` -> Service request_id
- bodyにduplicate request_idを持たせない
- canonical HTTP error mapping 200/201/400/401/403/404/409/413/500
- retry_input_unavailable -> 400 or 409ではなく`11`のcanonical domain mappingに従う（MVPでは400 validation/domain contract error）

## 6. SecretGuard

SecretsProvider:

- non-empty str success
- empty str -> `secret_value_invalid`
- bytes/number/object -> `secret_value_invalid`
- missing -> `secret_not_found`

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

## 7. Scheduling / Claim

- Workflow priority
- Job priority
- Dynamic order/source
- pool routing
- one internal running / Workflow Run
- multiple Workflow Runs parallel
- pause/resume
- non-preemptive priority update
- internal claim exactly-one race
- External claim同ordering
- concurrency group `Foo` と `foo` は別group

## 8. Runner / IPC

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

## 9. Timeout

- internal no-timeout long Action
- internal timeout -> cancel/terminate -> retry
- external/human/reusable timeout reject

## 10. Dynamic Jobs

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

## 11. Reusable Workflow

- registered ID
- relative path caller source directory基準
- traversal/symlink escape
- non-filesystem caller relative reject
- binding固定 / Retry same binding
- Parent/Child Output/state isolation
- cycle
- direct Child control reject
- Dynamic+Reusable
- restart duplicate防止
- Child Workflow Output spill

## 12. Managed ArtifactStore

- put_file durable copy
- source work_dir traversal reject
- immutable same-name generations
- managed materialize current work_dir
- materialize destination safety
- Retry current generation
- managed retention deletes data
- store finalize後DB failure -> orphan cleanup/no metadata exposure

External Artifact:

- URI metadata register
- no fetch
- Core retention external data非削除
- External LLM task_submit Artifactはexternal reference only

## 13. External LLM

- arbitrary JSON result
- large result spill
- activation one Task/Attempt
- candidate ordering / concurrent claim
- claim_next same selection
- lease requeue/fail
- stale/cancel submit reject
- Pause/recovery

## 14. Human

- pending/approve/reject
- Job Output null
- concurrent first-wins
- cancel/pause/retry
- timeout無し

## 15. Automatic / Manual Retry

- auto retry default off
- max-attempts / condition / backoff
- Retry target Input fixed
- failed Job with prior Attempt/Input Snapshot -> manual retry allowed
- activation前failure with no Attempt/Input Snapshot -> `retry_input_unavailable`
- `retry_input_unavailable` はsame Run retryせずnew Workflow Run要求
- Service/API error mapping
- manual retry completed/failure Run reopen
- run_attempt++
- success/cancelled Run retry reject
- blocked/skipped descendants re-evaluate
- failed non-target descendant not auto-retried
- terminal Run not reopened by Recovery

## 16. Same-Run Result Reuse

Scope:

- same Workflow Runだけ
- cross-Run automatic reuse無し

Key:

- persistent Input digest
- direct upstream Artifact identity
- entire Definition hash
- executor identity/action version

Negative:

- Input/Artifact/Definition/Action version changed
- Payload missing/digest bad
- state.get使用 -> ineligible
- frozen dependency外Artifact dynamic materialize -> ineligible

Manual Retry descendant:

- success + key match -> existing success + `job_result_reused`
- mismatch/ineligible/Payload missing -> `successful_job_result_not_reusable`
- same Job Runへchanged InputのAttemptを自動生成しない
- new Workflow Run要求

RetentionによるPayload/managed Artifact欠落もsilent reuse禁止。

## 17. Recovery

- Parent restart old running -> runner_lost
- queued/waiting/backoff restore
- Dynamic expansion duplicate無し
- Child duplicate無し
- reuse_check_pending restore
- completed Run Recovery-only reopen無し

## 18. Persistence / Idempotency / Retention

- migration/WAL/FK/busy timeout
- unknown future schema version reject
- output inline/blob column constraints
- reuse context/key/eligible/pending
- internal running unique
- Dynamic/Reusable/External/Human unique
- state current/history atomic
- concurrency race/case-sensitive group
- FK NO ACTION / child-first explicit retention deletion
- idempotency Actor/AccessScope isolation
- TTL replay/conflict
- expired row replacement
- task_claim replay no extra Lease

## 19. Authorization / Security

- AllowAll/Deny/filtered scope
- all public read/write authorize
- Definition list/info authorization
- log path safety
- Artifact URI no auto-fetch
- arbitrary shell無し

## 20. Platform / CI

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

## 21. MVP completion gate

1. `01`〜`12`受入条件対応
2. dependency install/import Python3.10+ / Windows/Linux
3. migration/FK retention semantics
4. process integration Windows/Linux
5. claim/concurrency races
6. PayloadStore inline/spill/crash
7. Service/MCP/HTTP adapter contract
8. Managed ArtifactStore
9. External/Human E2E
10. Dynamic1000/nested/rollback
11. Reusable binding/cycle
12. SecretGuard + Secret value contract
13. same-run Result Reuse
14. Retry Input availability boundary
15. idempotency scope/TTL

WebUI E2Eは後続。
