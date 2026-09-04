# JobRunner 基本設計

- Status: Draft v1.5
- 対象: MVP
- WebUI: 画面構成のみ後続
- 用語: GitHub Actions に対応概念がある場合は可能な限り合わせる

## 1. 目的

JobRunner は既存アプリケーションへ組み込んで使う軽量な Persistent Workflow / Job Runtime。

提供:

- Workflow / Workflow Run / Job / Attempt / Step
- `needs`
- Action Registry / Validator Registry
- Runner / Runner Pool
- Retry / Resume / Pause / Cancel
- Run History
- JSON Input/Output inspection・Output透過永続化
- ArtifactStore / ArtifactRef
- Human Review
- External LLM pull execution
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log / Progress
- Workflow State current/history
- SQLite persistence
- MCP / HTTP / Python 共通Service

非目標:

- Airflow / Temporal / n8n級の独立大規模基盤
- Kubernetes / distributed scheduler
- 任意shell / 任意Python sourceの標準実行
- GUI Workflow editor
- CLI / Cron scheduler
- 通知connector
- 完全GitHub Actions互換
- 本格sandbox / OS resource manager
- 認証基盤

## 2. MVP Python / OSS

Python >=3.10。

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

- YAML = 1.2
- JSON Schema = Draft 2020-12
- Canonical JSON = `jobrunner.canonical-json.v1`
- Core typed modelはstrict/no-coercion
- SQLite / process / JSON / UUID / hashlib等はPython標準libraryを優先

## 3. Package / optional dependency

1 GitHub Repository / 1 Python Package。

```text
jobrunner/
├─ definitions/
├─ runtime/
├─ runners/
├─ actions/
├─ validators/
├─ expressions/
├─ persistence/
├─ artifacts/
├─ events/
├─ security/
└─ adapters/{mcp,web}/
```

Extras=`jobrunner`, `[mcp]`, `[web]`, `[all]`。

Core -> Adapter逆依存無し。Parent既存MCP/Web serverへregisterし、JobRunner専用serverを別管理しない。

## 4. 用語

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
| Artifact | 明示登録するimmutable成果物 |
| ArtifactRef | ArtifactをJSON data flowで参照する値 |

正式用語は`Runner`。`Worker`は使わない。

## 5. 組み込み / Process Bootstrap

Parent startupでRuntime/Runner Poolを自動起動。JobRunner専用service手動起動不要。

Integration Bootstrap role:

```text
parent
runner
action_runner
```

Windows `spawn` 前提。Callable/provider instanceを親memoryからpickle継承しない。

Action RunnerへDB/Auth/Secrets/Store credentialを渡さない。Attempt SecretはRunnerが必要値だけIPC注入。

## 6. Workflow Definition / reload / System baseline

Canonical authoring=YAML 1.2。Safe loader、duplicate/merge/custom tag/unknown key reject。Core typed modelは値を暗黙変換しない。

Job `executor`省略=internal、`uses`あり=reusable。

Root Workflow Run開始時に:

- source YAML / typed Definition / hash
- Workflow Input
- Action/Validator versions
- System workflow defaults snapshot
- effective runtime settings（default_runner_pool含む）
- effective Retention policy
- initial priority
- optional source_identity

を保存する。

System baselineにはRunner default、Dynamic上限、External lease、Output threshold、Log/Progress、Retention defaultsを含む。Root Run開始後にSystem configが変わっても既存Run lineageへ反映しない。

Reusable Childはcurrent System configを読み直さずParent RunのSystem baselineを継承し、Child Workflow settingsを重ねてeffective settings/Retentionをbindingへ固定する。

Workflow YAMLはparent restart無しでreload可能。`wf_start`は実行開始時にsource bytesを必ず再読込・validateして使用し、mtime/size cacheだけをSource of Truthにしない。Invalid new YAMLはold cacheへsilent fallbackせずnew Run開始拒否。Existing Runは元snapshot継続。

## 7. Priority / Concurrency

Root Workflow Run初期priority:

```text
wf_start.priority specified -> request value
otherwise -> Workflow Definition priority
```

Child Workflow Run priority=current root Workflow Run priority。

Root `wf_priority_update`はroot + 全non-terminal descendant Child Runへ同値を伝播。Future Childもupdated root priorityを継承。Running Job preempt無し。Child direct priority updateは禁止。

Concurrency scopeは:

```text
(workflow_id, resolved concurrency group)
```

別Workflow IDの同名groupは競合しない。Paused holderはslotを保持する。Definition更新で同じscopeのRun間に`max-runs`差がある場合は既存holderを破らない保守的capacityを使う。

Internal scheduling軸:

1. Workflow Run priority
2. Job priority
3. Dynamic order rank
4. source order
5. Job ready_at
6. opaque Job Run ID

External Task claimは5番目に`external_tasks.available_at`を使い、Job `ready_at`は使わない。

## 8. Expression / Input / Secrets

`${{ ... }}` はGitHub Actions風。

- CEL: condition/value
- JMESPath: JSON filter/projection
- Custom Validator: trusted parent callable

Core context shape:

```text
workflow = {id,name,version}
run      = {id}
job      = {template_id,key,executor}
```

Dynamic template expansion中は`job.key=null`。記載外fieldを暗黙公開しない。

Job Input activation snapshot:

```text
persistent_input
secret_bindings
input_digest
```

Secret値は保存しない。MVP Secret参照はinternal Job `with` の1 scalar全体だけ。

Unique Secret名はAttempt開始時に1回だけ解決し、そのAttempt内のbinding・redaction・Artifact scanで同じ値を使う。Secret bindingが1件でもある成功Attemptは、Secret値同一性を証明できないため自動Result Reuse不可。

Validatorはinternal/externalのみ。Human/Reusable parentでは不可。

## 9. Action / Validator Registry

MVP Registryは1 IDにつきcurrent version 1つ。

Action registrationは明示的にRuntime Handle利用有無を持つ。

```text
register_action(id, version, callable, uses_runtime=false)
uses_runtime=false -> action(execution_input)
uses_runtime=true  -> action(execution_input, runtime_handle)
```

Coreはcallable signatureからRuntime Handle要否を推測しない。Actionはsync/async/awaitable return対応。ValidatorはMVPでは同期・軽量callable `(result, persistent_input)` のみ。

Run start時version snapshot。Retry/Resume/Runner実行はcurrent Registry versionとのexact一致必須。Multi-version RegistryはMVP外。Version更新忘れは親責任。

## 10. Runner Pool / Runner / IPC

Poolは事前登録。`runs-on`省略時はRun snapshot `default_runner_pool`。

Pool config=Runner数 + routing + restart/liveness。Action allow-list、resource quota、sandbox、Pool global pause無し。

Same Workflow Run internal Job同時最大1。別Run並列可。

Runner管理ProcessとAction Runner子Process分離。

Default:

```text
heartbeat=5s
runner_lost=20s
main_loop_stale=15s
```

IPC=JSON Lines v1。Handshake=`ready -> start -> result|error -> exit`。stdout/stderrはExecution Log。

Cancel/timeout terminalizationが先に成立した後のlate Action resultは採用しない。Timeout graceはcleanup猶予でありdeadline後の成功Result受理猶予ではない。

stdout/stderrはstreaming Secret redactionを行い、chunk境界をまたぐ既知Secretもraw保存しない。

## 11. Readiness / Retry pending snapshot

```text
queued + ready_at=NULL     -> dependency/condition待ち
queued + ready_at!=NULL    -> 次Attempt用Input snapshot確定済み
```

Internal初回activationはpending snapshotをJob rowへ保存しRunner claimでAttemptへcopy。

External/Human/Reusable初回は直接Attempt。

Automatic/Manual Retryは全executorでfailed Attempt Input/bindings/digestをpending snapshotへcopyし、次Attemptは実行開始時に作る。Resume/Recoveryでready済みInputを再評価しない。

## 12. Maintenance / Timeout

Maintenance Loop=内部deadline/housekeeping専用。Cron Schedulerではない。

対象=Retry due、External Lease expiry、concurrency wake、Retention、orphan/idempotency cleanup。

Default max sleep5秒、busy loop無し。

`timeout-minutes`はinternalのみ。Hidden default無し。

Orphan filesystem cleanupにはSystem housekeeping設定 `orphan_cleanup_grace_seconds` を使う。既定300秒、finite > 0。Workflow Run semanticsではないためRun snapshot対象外。

## 13. 状態 / 条件

Job status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Workflow Run status=`queued|running|paused|completed`、Conclusion=`success|failure|cancelled`。

Dynamic group status=`queued|running|completed`。

`if` default=`success()`。Skipped dependencyはeffective successではなく後段default skip。Cancel後`always()`でもnew activation無し。

## 14. JSON Output / Result validation

Internal/External result=任意JSON-compatible value。

Result validation:

1. canonical JSON
2. optional JSON Schema
3. optional Validator
4. optional success_if
5. SecretGuard
6. PayloadStore

Default inline threshold=4MiB。超過はfilesystem spillで、max Outputではない。

Human approve Output=`null`、Human `outputs.schema`は禁止。

Reusable ParentはChild Workflow Output objectをresultとし、optional Parent `outputs.schema`を検証してParent Attempt Outputとして保存する。

## 15. ArtifactStore / ArtifactRef

Managed Artifact=Core Store管理。External Reference=URI metadataのみ。

Artifact digestのMVP形式は `sha256:<lowercase hex64>`。ManagedはCore計算、External Referenceは指定時に形式だけ検証する。

Cross-run Managed Artifact利用は:

- ArtifactRefがcurrent persistent Inputへ明示存在
- source data存在
- Authorization許可

が必須。Raw IDのみ不可。

Cross-run ArtifactRefはsource Retentionをpinしない。

## 16. Dynamic Job

Dynamic templateは仮想groupでありtemplate自身のJob Run/Attempt/Retry/Progress単位無し。Generated concrete Jobだけ`job_runs`へ入る。

Root=`foreach`、Nested=`foreach.parent/items`。

- depth固定上限無し
- generated maxはWorkflow Run snapshot `effective_settings.max_dynamic_jobs`、canonical default1000
- all-or-nothing expansion
- stable key推奨
- `key`省略時は0-based index fallbackし、各expansionで `dynamic_index_key_fallback` Eventを1回記録
- order_by対応
- `if=false` skip と `foreach=[]` successを区別
- expansion digestでManual Retry後のgenerated set不変性を検証

## 17. Reusable Workflow

Parent Job first activationでChild source bytesを再読込・validateしbinding固定:

- Child Definition
- Child Action/Validator versions
- inherited System baseline
- Child effective settings
- Child Retention policy

Retryはsame binding。Nested Childも同じRoot lineage System baseline。

Child自身のConcurrencyはChild `workflow_id + group` scopeで通常Runと同じAdmissionを行う。QueueならChild Runを作ってParentは`waiting_child`、RejectならChild Runを作らずParent Attemptをfailureにする。

Parent/Child state非共有。ArtifactはArtifactRefで明示mapping。Child Artifact自動mirror無し。

Child direct pause/resume/cancel/retry/priority update禁止。

## 18. External LLM / Human

External:

- Attempt + Task
- Job external override > Run effective lease setting
- fixed Lease、renew/heartbeat無し
- atomic claim/submit
- `task_submit(claim_next=true)`はsubmit + optional next claim + idempotency resultを同一transaction
- Lease requeue時はTask `available_at`を更新

Human:

- approve/rejectのみ
- no Lease/timeout/Validator/Output Schema
- approve Output null
- first-wins
- no success override/manual skip/rewrite

## 19. Retry / Same-Run Reuse

Retry=new Attempt。ただしrequest/schedule時点ではAttemptを作らずpending Input snapshotを固定。

- retry absent = auto retry無し/max1
- `retry:{}` = max2 / failure.retryable / zero backoff

Manual Retryはfailed Attempt/Input必須。Pre-Attempt failureはnew Run要求。

Completed failure Workflow RunをManual Retryでreopenする場合はConcurrency slotを再取得する。`on-limit=reject`でslot不可ならRun/Jobを変更せず失敗、queueなら`wait_reason=concurrency`でreopenする。Reopen commit時に過去Workflow Outputのcurrent pointerをclearし、再完了までRun Outputは unavailable とする。

Successful descendantはcurrent dependency contextから `if` / expected Input / Artifact / versions / stored Output validationを再確認。Mismatchならsame Runでchanged Input再実行せずnew Workflow Run要求。

Runtime Handleで `state.get` または `state.set` を使用したSuccessful Attempt、Secret binding付きAttempt、およびpersistent Input外ArtifactをmaterializeしたAttemptは自動Result Reuse不適格。

Dynamic expansionもdigest mismatchならnew Run要求。

## 20. Pause / Cancel

Pause=running internal継続、新claim/expansion停止、existing submit/review/Child継続、Lease/Retention継続。Concurrency holderはslotを保持する。

Cancel=new activation禁止、queued/waiting cancel、Lease invalidation、internal graceful cancel、Child propagation。Cancel terminalization後のlate resultは採用しない。

Public force-kill無し。

## 21. State / Log / Progress

State=get/set、persistent、last-write-wins、history。CAS/increment無し。

`state.set`は呼出時にcurrent+historyを即時commitし、Attemptが後でfailure/cancel/timeout/runner_lostになってもrollbackしない。業務上のtransactionが必要なら親Action側で実現する。

Execution Log=全executor Attempt共通file。Input/Output全bodyをCoreが自動dumpしない。Log verbosityはWorkflow Run effective settings snapshotを使う。

Event Log=append-only DB。

Progress:

- Job=`auto|explicit|none`。Job Run作成時にresolved modeをsnapshot
- Workflow=`auto|none`。Workflow Run effective settings snapshotを使う
- Workflow auto=concrete Job Run平均
- Dynamic templateは分母外
- Reusable ParentはChild progressを利用可能
- ProgressはConclusionに影響しない

Stepは観測単位のみ。start/finish metadataを別保存し、表示用telemetryの既知Secretはredactする。

Runtime Handleのstate/artifact resource operationもAuthorizationProvider hookを通す。

## 22. Persistence / Idempotency / Retention

Standard SQLite。MVP table=18。

重要:

- System baseline/effective settings snapshot
- one running internal / Run
- Dynamic expansion unique
- Child / Parent Attempt unique
- Task / Lease / Review unique
- pending/Attempt Input snapshot整合
- state current+history atomic
- canonical UTC timestamp=`YYYY-MM-DDTHH:MM:SS.ffffffZ`

Idempotency=completed resultのみ保存、reserved row無し。Commit transaction内でkey/hash再確認。Replayは初回result/HTTP status。

Retention=null=unlimited、System baseline -> Workflow override。Childはinherited System baseline -> Child override。Parent run-history expiryはChild subtree実効上限。

Managed Artifact dataはdata retention、metadata retention、Run history上限の早い方で削除可能。Metadata期限が先ならManaged dataを先に消してからmetadata rowを削除する。External Referenceの外部実体は削除しない。

## 23. Authorization / Service / MCP / HTTP

Authentication親責任。Default AllowAllだがhook省略無し。

Canonical logical API:

```text
wf_definition_list/info
wf_start
wf_run_list/info
wf_pause/resume/cancel/retry/priority_update
wf_input_info/read
wf_output_info/read
wf_state_list/read/history
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_event_list
wf_runner_info
```

`wf_run_info(include_jobs=true)` はconcrete JobsとDynamic group summaryを別配列で返す。Attempt/Step/Artifactをoptional include可能。External/Human/Reusable Jobはactive Task/Review/Child Run IDを辿れる。

Input本文は専用read。Job/Attempt persistent InputのSecretはreference stringのままでmaterialized valueを返さない。

Workflow State public APIはread-only。State mutationはRuntime Handle/親内部Service経由だけ。

存在しないAPI=Lease renew/heartbeat、manual Job skip/success override、Review rewrite。

MCP=`<namespace>_wf_*`。HTTP prefix=`/api/jobrunner/v1`。Core Service modelはstrict/no-coercionで、HTTP query文字列だけAdapterが明示parseする。

## 24. Scheduler / CLI / WebUI

User Workflow Scheduler/Cron/CLIはMVP無し。

WebUI画面構成のみ後続。Service/HTTP contractは固定済み。

## 25. Source of Truth

実装時は`docs/detailed-design/01`〜`13`をSource of Truthとする。衝突時は最新の具体的詳細設計を優先し、基本設計も同期する。
