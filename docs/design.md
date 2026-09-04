# JobRunner 基本設計

- Status: Draft v0.9
- 対象: MVP
- WebUI: 画面設計のみ後続
- 用語: GitHub Actions に対応概念がある場合は可能な限り合わせる

## 1. 目的

JobRunner は既存アプリケーションへ組み込んで使う軽量な Persistent Workflow / Job Runtime。

提供:

- Workflow / Workflow Run
- Job / Attempt / Step
- `needs`
- Action Registry / Validator Registry
- Runner / Runner Pool
- Retry / Resume / Pause / Cancel
- Run History
- JSON Output透過永続化
- ArtifactStore / Artifact参照
- Human Review
- External LLM pull execution
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log
- SQLite persistence
- MCP / HTTP / Python 共通Service

非目標: Airflow/Temporal/n8n級の独立大規模基盤、Kubernetes/distributed scheduler、任意shell標準機能、GUI workflow editor、CLI/Cron、通知、完全GitHub Actions互換、本格sandbox、認証基盤。

## 2. MVP Python / OSS

Python >=3.10。

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

`cel-python`はPyPI `cel-python` / cloud-custodian implementation。ライセンスは `ruamel.yaml/pydantic/jsonschema/jmespath=MIT`, `cel-python=Apache-2.0`。

SQLite/process/JSON/UUID等はPython標準libraryを優先する。

## 3. 用語

| 用語 | 意味 |
| --- | --- |
| Workflow | YAMLの設計図 |
| Workflow Run | Workflowを1回起動した実行実体 |
| Job | Workflow内の処理単位 |
| Attempt | Retryを含むJobの各試行 |
| Step | Job内の観測/ログ/進捗単位 |
| Action | 親Registryに登録する実処理 |
| Validator | Job resultの親定義業務検証 |
| Runner | internal Jobを実行する常駐Process |
| Runner Pool | Runner group |
| Artifact | Actionが明示登録するimmutable成果物 |

正式用語は`Runner`。`Worker`は使わない。

## 4. 組み込み構成

```text
Parent System
├─ JobRunner Runtime / Service
├─ Action Registry
├─ Validator Registry
├─ SQLite
├─ PayloadStore
├─ ArtifactStore
├─ Maintenance Loop
├─ MCP Adapter optional
├─ HTTP/Web Adapter optional
└─ Runner Pools
   └─ Runner Process
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
```

親起動時にRuntime/Registry/Runner Poolを自動初期化。JobRunner専用サービス手動起動不要。

Maintenance LoopはCron/SchedulerではなくRetry/Lease等の期限処理専用。

## 5. Workflow Definition

Canonical format YAML。safe loader、duplicate/merge/custom tag/unknown key reject。

Workflow Inputは`type/required/nullable/default/schema`。null許可は`nullable:true`。

`env`はJSON-compatible literal-only。Expression/Secret禁止。

Concurrency groupはnon-empty string、case-sensitive完全一致、暗黙normalization無し。

Run開始時snapshot:

- Workflow ID/version/source YAML/definition hash
- Input
- Action ID/version
- Validator ID/version
- optional non-empty `source_identity`

既存Runは元YAML変更の影響無し。

## 6. Expression / Validation / Secret

GitHub Actions風`${{ ... }}`。

- CEL: condition/value計算
- JMESPath: JSON filter/projection
- Custom Validator: 親定義trusted lightweight callable

Result validation order:

1. JSON-compatible/canonical
2. optional JSON Schema
3. optional Custom Validator
4. optional `success_if`
5. SecretGuard
6. PayloadStore

Validatorはinternal/external Jobでoptional。Human/Reusable parent Jobでは不可。Validatorはresultを変換せず、persistent Inputだけを参照しSecret/Runtime Handleを受け取らない。重い検証はnormal Jobへ分離。

Secretsはinternal Action Job `with`だけ。Secret valueはnon-empty string。

Known Secret:

- Output/State/Artifact metadata/Event/error persistence reject
- managed Artifact exact byte sequence保存前scan
- Log/stdout/stderr redact

変形Secretの完全検出は保証しない。

## 7. Action / Validator Registry

両Registryは親bootstrapで各Process起動時に再構築。

Identity:

```text
action_id + action_version
validator_id + validator_version
```

全てnon-empty string。Version更新は親責任。

Action sync/async対応。親Actionは`ActionFailure(code,message,retryable,details)`でstructured failureを返せる。

Runtime Handle:

- progress / step / log
- state get/set
- cancel check
- managed Artifact put/materialize
- external Artifact reference

## 8. Runner / Runner Pool

Pool事前登録。未登録Run開始前error。`runs-on`省略はSystem `default_runner_pool`、default文字列`default`。

同一Workflow Run internal Job同時最大1。別Run並列可。

選択:

1. Workflow priority
2. Job priority
3. Dynamic order
4. source order
5. ready time
6. deterministic ID

Priority signed64。External claimも同軸。

## 9. Runner / IPC / Heartbeat

Runner管理ProcessとAction子Process分離。Heartbeat Thread別。

Default heartbeat5秒 / lost20秒。SupervisorがOS exit + heartbeatを監視しrunner_lostをexactly once Recovery。

Runner↔Action Runnerは専用JSON Lines、stdout/stderrはLog。

Large Action Returnはworkdir reserved `result.json`へ書き、IPCはpath/size/digestのみ。

Parent正常shutdownはWorkflow cancelではない。未完了running Attemptは次回runner_lost Recovery。

## 10. Maintenance Loop / Timeout

Maintenance Loopはbusy-loopせずdeadline/event待機し、少なくとも:

- `retry_not_before`
- External Lease `expires_at`
- concurrency wake

を処理。Default max sleep5秒。Pause中もLease expiryは進む。Restart時overdue deadlineを先処理。

Job `timeout-minutes`はinternalだけ、hidden default無し。ExternalはLease、Human期限無し、ReusableはChild内timeout。

## 11. 状態 / 条件

Job status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Workflow Run status:

```text
queued|running|paused|completed
```

Conclusion:

```text
success|failure|cancelled
```

`if` default=`success()`。

Effective success = success または failure+continue-on-error。

Normal skipped dependencyはeffective successではなく後段default skip。Dynamic groupはaggregateでsuccessになり得る。

Cancel後はalwaysでもnew activation無し。

## 12. JSON Output / PayloadStore

Action/External resultは任意JSON-compatible value。

Workflow top-level outputsはname mappingなので全体object。

Default inline threshold=4MiB。

- <= threshold SQLite
- > threshold filesystem PayloadStore

4MiBはmaxではない。Storage kind透過。

Blob size/digest integrity。missing/corruption fail-closed。

## 13. ArtifactStore

ArtifactはOutputと別の明示成果物。

Managed:

- `runtime.artifact.put_file`
- temp自動Artifact化無し
- retry same name=new generation
- materialize可
- retentionでStore data delete

External reference:

- URI metadata
- Core fetch/delete無し
- External LLM artifactはreferenceのみ

Standard `LocalArtifactStore`。

## 14. Dynamic Job

Root foreach / Nested `foreach.parent/items`。

- parent DAG edge
- fixed depth limit無し
- generated max default1000/Run
- all-or-nothing
- stable key推奨
- full logical key parent path込み
- filesystem opaque ID
- expansion preflightでAction/Validator version検証

## 15. Reusable Workflow

`uses`でChild Run。

Relative fileはcaller source directory基準、Workflow root escape reject。Non-filesystem callerはregistered ID。

Parent Job first activationでbinding固定:

- Child Definition hash
- Child Action versions
- Child Validator versions

Retry same binding。Missing snapshot versionはfail-closed。

Parent/Child state非共有。Child direct public control禁止。

## 16. External LLM / Human

External:

- activation Attempt+Task
- Lease default60分、expiry default requeue
- Maintenance Loopがtraffic無しでもexpiry処理
- atomic claim/owner submit
- optional Custom Validator
- arbitrary JSON PayloadStore
- claim_next same ordering

Human:

- pending Review
- approve/reject
- no Lease/timeout/Validator
- first-wins
- Job Output null

## 17. Retry / Recovery

Retry=new Attempt、persistent Input/Definition/Action+Validator version/Dynamic iteration/Reusable binding固定。

Retry block absent=Automatic Retry disabled/max1。

`retry:{}` canonical default:

```text
max-attempts=2
if=failure.retryable
backoff=0
```

Backoff configurable finite values。Jitter MVP無し。

Core retryable default:

- true: runner_lost, job_timeout, external_lease_expired(fail), payload_storage_failed
- false: unhandled Action/process/protocol/schema/validator exception/version/human/security/reuse/input unavailable等
- ActionFailure/Validator failureは親指定retryableを使用

Manual Retryはfailed Job + prior failed Attempt/Input Snapshot必須。Pre-Attempt failureは`retry_input_unavailable`でnew Run要求。

Completed/failure Run manual retryはsame Run reopen + run_attempt++。Recoveryだけでcompleted reopen無し。

## 18. Same-Run Result Reuse

Automatic reuse same Workflow Runのみ。

Key:

- persistent Job Input
- direct upstream Artifact identities
- whole Definition hash
- executor/Action identity
- Validator identity

state.getやundeclared Artifact materializeはineligible。

Manual Retry後successful descendantsはkey再検証。Mismatch/ineligible/Payload/version問題は`successful_job_result_not_reusable`でnew Run要求。

## 19. Pause / Cancel

Pause:

- running internal継続
- new internal/External claim/Dynamic expansion停止
- existing submit/review/Child進行可
- Lease expiry継続

Cancel:

- new activation禁止
- queued/waiting cancel
- Lease invalidation
- internal graceful cancel
- Child propagation

## 20. Workflow State / Log

State persistent get/set, last-write-wins, revision/history。CAS/increment無し。Child independent。

Execution Log=file、Event Log=append-only DB、Step=観測単位。

Temp終了時削除、sandboxではない。

## 21. Persistence / Idempotency

SQLite WAL/FK/busy timeout。FK NO ACTION、Retention explicit delete order。

重要:

- one running internal/Run
- Dynamic expansion unique
- Child/Parent Attempt unique
- Task/Lease/Review unique
- state current+history atomic
- deadline indexes
- concurrency case-sensitive
- Payload integrity
- Action/Validator snapshot/reuse metadata

Idempotency scope=namespace+resource+AccessScope+Actor/client principal、TTL24h。Expired row replace可能。

HTTP replayは初回status/bodyを維持するためadapter replay metadataを保存可能。

## 22. Authorization / Service / MCP / HTTP

Authentication親責任。全public read/write Authorization、default AllowAll。

Canonical logical API:

```text
wf_definition_list/info
wf_start
wf_run_list/info
wf_pause/resume/cancel/retry/priority_update
wf_output_info/read
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_runner_info
```

No ambiguous `wf_list/wf_info` aliases。

MCP public `<namespace>_wf_*`。Namespace collision reject。

HTTP standard prefix `/api/jobrunner/v1`、exact contractは`11`。Workflow Definition infoはquery `workflow_ref`。State change idempotencyは`Idempotency-Key` header。

MVP HTTP statusは200/201/400/401/403/404/409/413/500。422無し。Idempotent replayはoriginal status/body。

Run infoにOutput/Log本文無し。Output readはJMESPath select可。MCP oversized full resultはno truncate error。

## 23. Retention / Scheduler / CLI / WebUI

Retention default unlimited。

User Workflow Scheduler/Cron/CLIはMVP無し。Maintenance Loopは内部deadline処理でありScheduler機能ではない。

WebUI画面構成だけ後続。Service/HTTP contractは固定済み。

## 24. Source of Truth

実装時は`docs/detailed-design/01`〜`13`がSource of Truth。衝突時は最新具体的詳細設計を優先し、基本設計も同期する。
