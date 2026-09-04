# JobRunner 基本設計

- Status: Draft v1.6
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

Integration Bootstrap role=`parent|runner|action_runner`。

Windows `spawn` 前提。Callable/provider instanceを親memoryからpickle継承しない。

Action RunnerへDB/Auth/Secrets/Store credentialを渡さない。Attempt SecretはRunnerが必要値だけIPC注入。

## 6. Workflow Definition / Resolver / reload / System baseline

Canonical authoring=YAML 1.2。Safe loader、duplicate/merge/custom tag/unknown key reject。Core typed modelは値を暗黙変換しない。

Job `executor`省略=internal、`uses`あり=reusable。

**MVPで実行可能な全Workflow参照はResolverからUTF-8 YAML source bytesを取得できることが必須。** Registered Workflow IDもtyped objectだけではなくsource bytesを持つ。Relative reusable referenceだけfilesystem base directoryを必要とする。

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

Reusable ChildはParent RunのSystem baselineを継承し、Child settings/Retentionをbindingへ固定する。

Workflow YAMLはparent restart無しでreload可能。`wf_start`は実行開始時にsource bytesを必ず再取得・validateし、mtime/size cacheだけをSource of Truthにしない。Reusable first bindingもcurrent Child source bytesを同様に読む。Invalid/unavailable sourceはold cacheへsilent fallbackしない。Existing Run/既存bindingは元snapshot継続。

## 7. Priority / Concurrency / Run status

Root Workflow Run初期priority:

```text
wf_start.priority specified -> request value
otherwise -> Workflow Definition priority
```

Child Workflow Run priority=current root Workflow Run priority。Root `wf_priority_update`はroot + 全non-terminal descendant Child Runへ伝播。Running Job preempt無し。Child direct update禁止。

Concurrency scope=`(workflow_id, resolved group)`。別Workflow IDの同名groupは競合しない。Definition更新で同scopeの`max-runs`が異なる場合は既存holderの厳しい上限を破らない保守的capacityを使う。

Workflow Run statusのMVP意味:

```text
running   = admission済みnon-terminal Run
queued    = Concurrency slot待ち専用
paused    = pause済み。wait_reason=concurrencyならwaiterでslot無し、それ以外はadmitted holderでslot保持
completed = conclusion確定
```

Pauseしたadmitted holderはslot保持。PauseしたConcurrency waiterはwake候補外で、Resumeするとqueued/concurrencyへ戻る。

Internal scheduling軸:

1. Workflow Run priority
2. Job priority
3. Dynamic order rank
4. Dynamic/source order
5. Job ready_at
6. stable Job `job_key`
7. opaque Job Run ID

Static JobはYAML declaration orderをScheduling保証にせず`source_order=0`、同条件では`job_key`で安定tie-break。External Task claimは5番目に`external_tasks.available_at`を使い、同様に`job_key`をopaque IDより先に使う。

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

Job Input activation snapshot=`persistent_input + secret_bindings + input_digest`。

Secret値は保存しない。MVP Secret参照はinternal Job `with` の1 scalar全体だけ。

Unique Secret名はAttempt開始時に1回だけ解決し、そのAttempt内のbinding・redaction・Artifact scanで同じ値を使う。Secret bindingが1件でもある成功AttemptはSecret値同一性を証明できないため自動Result Reuse不可。

Validatorはinternal/externalのみ。Human/Reusable parentでは不可。

## 9. Action / Validator Registry

MVP Registryは1 IDにつきcurrent version 1つ。

```text
register_action(id, version, callable, uses_runtime=false)
uses_runtime=false -> action(execution_input)
uses_runtime=true  -> action(execution_input, runtime_handle)
```

CoreはsignatureからRuntime Handle要否を推測しない。Actionはsync/async/awaitable return対応。ValidatorはMVPでは同期・軽量 `(result, persistent_input)` のみ。

Run start時version snapshot。Retry/Resume/Runner実行はcurrent Registry versionとのexact一致必須。Multi-version RegistryはMVP外。Version更新忘れは親責任。

## 10. Runner Pool / Runner / IPC

Poolは事前登録。`runs-on`省略時はRun snapshot `default_runner_pool`。

Pool config=Runner数 + routing + restart/liveness。Action allow-list、resource quota、sandbox、Pool global pause無し。

Same Workflow Run internal Job同時最大1。別Run並列可。

Runner管理ProcessとAction Runner子Process分離。

Default=`heartbeat 5s / runner_lost 20s / main_loop_stale 15s`。

IPC=JSON Lines v1。Handshake=`ready -> start -> result|error -> exit`。stdout/stderrはExecution Log。

Cancel/timeout terminalizationが先ならlate Action resultを採用しない。Timeout graceはcleanup猶予であり成功猶予ではない。

stdout/stderrはraw bytes段階でstreaming Secret redactionし、chunk境界をまたぐ既知Secretもraw保存しない。

## 11. Readiness / Retry pending snapshot

Concrete Job:

```text
queued + ready_at=NULL     -> dependency/condition待ち
queued + ready_at!=NULL    -> 次Attempt用Input snapshot確定済み
```

Internal初回activationはpending snapshotをJob rowへ保存しRunner claimでAttemptへcopy。External/Human/Reusable初回は直接Attempt。

Automatic/Manual Retryは全executorでfailed Attempt Input/bindings/digestをpending snapshotへcopyする。Retry pendingへ戻すJobは`status=queued, conclusion=NULL, completed_at=NULL`。次Attemptは実行開始時に作る。Resume/Recoveryでready済みInputを再評価しない。

## 12. Maintenance / Timeout

Maintenance Loop=内部deadline/housekeeping専用。Cron Schedulerではない。

対象=Retry due、External Lease expiry、concurrency wake、Retention、orphan/idempotency cleanup。

Default max sleep5秒、busy loop無し。

`timeout-minutes`はinternalのみ。Hidden default無し。

Orphan cleanup=`orphan_cleanup_grace_seconds` default300、finite > 0。Operational housekeepingでRun snapshot対象外。

## 13. 状態 / 条件

Concrete Job status=`queued|running|waiting_external|waiting_review|waiting_child|completed`。

Conclusion=`success|failure|cancelled|skipped|blocked`。

Dynamic group status=`queued|running|completed`。

`if` default=`success()`。Skipped dependencyはeffective successではなく後段default skip。Cancel後`always()`でもnew activation無し。

## 14. JSON Output / Result validation

Internal/External result=任意JSON-compatible value。

Validation=`canonical JSON -> optional JSON Schema -> optional Validator -> optional success_if -> SecretGuard -> PayloadStore`。

Default inline threshold=4MiB。超過はfilesystem spillでmax Outputではない。

Human approve Output=`null`、Human `outputs.schema`は禁止。

Reusable ParentはChild Workflow Output objectをresultとし、optional Parent `outputs.schema`を検証してParent Attempt Outputとして保存する。

## 15. ArtifactStore / ArtifactRef

Managed Artifact=Core Store管理のimmutable data。

External Reference Artifact=URI metadataのみだが、**JobRunner上は論理immutable**。親が外部実体内容を変える場合は新しいArtifact ID/generationを登録する。Coreは外部実体不変性を検証しないため、破った場合は親責任。

Artifact digest形式=`sha256:<lowercase hex64>`。ManagedはCore計算、Externalは指定時に形式のみ検証。

Cross-run Managed Artifact利用はexplicit ArtifactRef + source data存在 + Authorizationが必須。Raw IDのみ不可。Cross-run ArtifactRefはsource Retentionをpinしない。

## 16. Dynamic Job

Dynamic templateは仮想groupで、template自身のJob Run/Attempt/Retry/Progress単位無し。Generated concrete Jobだけ`job_runs`へ入る。

Root=`foreach`、Nested=`foreach.parent/items`。

- depth固定上限無し
- generated max=Run snapshot `max_dynamic_jobs`、default1000
- all-or-nothing expansion
- stable key推奨
- key省略時0-based index fallback + 1 warning Event/expansion
- order_by対応
- `if=false` skip と `foreach=[]` successを区別
- expansion digestでManual Retry後のgenerated set不変性を検証

## 17. Reusable Workflow

全実行可能Workflow参照はResolverからYAML source bytesを取得可能であることが前提。

Parent Job first activationでcurrent Child source bytesを再取得・validateしbinding固定:

- Child Definition
- Child Action/Validator versions
- inherited System baseline
- Child effective settings
- Child Retention policy

Retryはsame bindingでChild sourceを再読込しない。

Child ConcurrencyはChild `workflow_id + group` scope。Admission成功ならChild `running`、QueueならChild `queued/concurrency` + Parent `waiting_child`、RejectならChild rowを作らずParent Attempt failure。

Parent/Child state非共有。ArtifactはArtifactRefで明示mapping。Child Artifact自動mirror無し。Child direct pause/resume/cancel/retry/priority update禁止。

## 18. External LLM / Human

External:

- Attempt + Task
- Job external override > Run effective lease setting
- fixed Lease、renew/heartbeat無し
- claim順はTask `available_at`を使用
- Lease requeueで`available_at`更新
- atomic claim/submit
- `task_submit(claim_next=true)`はsubmit + optional next claim + idempotency resultを同一transaction
- `task_info`は他claimantのlease_idを公開しない。SubmitはLease ID + claimant ownershipを再確認

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

Completed failure Run reopen時はConcurrency slot再取得。Rejectなら変更無し、QueueならRun `queued/concurrency`、Admissionなら`running`。Reopen commitでRun conclusion/completed/failureとcurrent Workflow Outputをclearする。

Successful descendantはcurrent dependency contextから`if` / expected Input / Artifact / versions / stored Output validationを再確認。Mismatchならsame Runでchanged Input再実行せずnew Workflow Run要求。

`state.get/state.set`使用Successful Attempt、Secret binding付きAttempt、persistent Input外Artifact materialize Attemptは自動Result Reuse不適格。

Dynamic expansionもdigest mismatchならnew Run要求。

## 20. Pause / Cancel

Admitted Run Pauseはrunning internal継続、新claim/expansion停止、existing submit/review/Child継続、Lease/Retention継続、Concurrency slot保持。

Concurrency waiter Pauseはslot無しでwake対象外。Resumeでqueued/concurrencyへ戻る。

Cancel=new activation禁止、queued/waiting cancel、Lease invalidation、internal graceful cancel、Child propagation。Cancel terminalization後のlate resultは採用しない。

Public force-kill無し。

## 21. State / Log / Progress / Step

State=get/set、persistent、last-write-wins、history。CAS/increment無し。

`state.set`は呼出時にcurrent+historyを即時commitし、Attemptが後でfailure/cancel/timeout/runner_lostになってもrollbackしない。

Execution Log=全executor Attempt共通file。Input/Output全bodyをCoreが自動dumpしない。Log verbosityはRun effective settings snapshot。

Event Log=append-only DB。

Progress:

- Job=`auto|explicit|none`、Job Run作成時snapshot
- Workflow=`auto|none`、Run effective settings snapshot
- Workflow auto=concrete Job平均
- Dynamic templateは分母外
- Reusable ParentはChild progress利用可
- ProgressはConclusionに影響しない

Stepは観測単位のみ。**MVPでは1 Attempt同時open Step最大1**。start/finish metadataを別保存し、`state.set` historyはその時点のopen StepまたはNULLへ紐付ける。表示telemetryの既知Secretはredactする。

Runtime Handleのstate/artifact resource operationもAuthorizationProvider hookを通す。

## 22. Persistence / Idempotency / Retention

Standard SQLite。MVP table=18。

重要:

- System baseline/effective settings snapshot
- Workflow Run queued/running/paused + concurrency holder/waiter invariant
- one running internal / Run
- one open Step / Attempt
- Dynamic expansion unique
- Child / Parent Attempt unique
- Task / Lease / Review unique
- pending/Attempt Input snapshot整合
- state current+history atomic
- canonical UTC timestamp=`YYYY-MM-DDTHH:MM:SS.ffffffZ`

Idempotency=completed resultのみ保存、reserved row無し。Commit transaction内でkey/hash再確認。Replayは初回result/HTTP status。

Retention=null=unlimited、System baseline -> Workflow override。Childはinherited System baseline -> Child override。Parent run-history expiryはChild subtree実効上限。

Managed Artifact dataはdata retention、metadata retention、Run history上限の早い方で削除可能。Metadata期限が先ならManaged dataを先に消してmetadata rowを削除。External Referenceの外部実体は削除しない。

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

Workflow State public APIはread-only。State mutationはRuntime Handle/親内部Service経由だけ。

存在しないAPI=Lease renew/heartbeat、manual Job skip/success override、Review rewrite、public State mutation。

MCP=`<namespace>_wf_*`。HTTP prefix=`/api/jobrunner/v1`。Core Service modelはstrict/no-coercionでHTTP query文字列だけAdapterが明示parseする。

## 24. Scheduler / CLI / WebUI

User Workflow Scheduler/Cron/CLIはMVP無し。

WebUI画面構成のみ後続。Service/HTTP contractは固定済み。

## 25. Source of Truth

実装時は`docs/detailed-design/01`〜`13`をSource of Truthとする。衝突時は最新の具体的詳細設計を優先し、基本設計も同期する。
