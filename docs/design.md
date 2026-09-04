# JobRunner 基本設計

- Status: Draft v1.2
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
- JSON Output透過永続化
- ArtifactStore / ArtifactRef
- Human Review
- External LLM pull execution
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log / Progress
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
└─ adapters/
   ├─ mcp/
   └─ web/
```

Extras:

```text
jobrunner
jobrunner[mcp]
jobrunner[web]
jobrunner[all]
```

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

親システムはimport可能なIntegration Bootstrapを1つ提供し、Processごとに再実行する。

```text
role=parent
role=runner
role=action_runner
```

- parent: Action/Validator Registry、AuthorizationProvider、SecretsProvider、optional Store factory
- runner: Registry metadata/Validator/Auth/Secrets/Storeをprocess-local再構築
- action_runner: Action callable解決のみ必須

Windows `spawn` 前提。Callable/provider instanceを親memoryからpickle継承しない。

Action RunnerへDB/Auth/Secrets/Store credentialを渡さない。Attempt SecretはRunnerが必要値だけIPC注入。

## 6. Workflow Definition / reload / settings snapshot

Canonical authoring=YAML 1.2。

- safe loader
- duplicate/merge/custom tag/unknown key reject
- Input=`type/required/nullable/default/schema`
- `env` literal-only
- Concurrency group=non-empty case-sensitive string
- Job `executor`省略時=internal
- `uses`があればresolved executor=reusable

System runtime defaultの主項目:

```text
default_runner_pool = default
max_dynamic_jobs = 1000
external_lease_minutes = 60
external_on_lease_expiry = requeue
output_inline_threshold_bytes = 4194304
execution_log_level = normal
workflow_progress_mode = auto
job_progress_mode = auto
idempotency_ttl_hours = 24
```

Workflow `settings` は許可項目だけSystem設定をoverride。External Jobはlease/expiryだけJob単位override可。

Workflow Run開始時に少なくとも:

- source YAML / typed Definition / Definition hash
- Workflow Input
- Action/Validator versions
- effective runtime settings（`default_runner_pool`含む）
- effective Retention policy
- optional `source_identity`

をsnapshot。

Workflow YAMLはparent restart無しでreload可能。Invalid new YAMLはold cacheへsilent fallbackせずnew Run開始を拒否。Existing Runは元snapshot継続。

Action/Validator Python hot reload専用機構は作らず、親development autoreload/Runner recycleへ任せる。

## 7. Expression / Input / Validation / Secrets

`${{ ... }}` はGitHub Actions風。

- CEL: condition/value計算
- JMESPath: JSON projection/filter
- Custom Validator: trusted lightweight parent callable

Job Inputはactivation時にpersistent snapshotを作る。

```text
persistent_input
secret_bindings
input_digest
```

Secret値そのものはsnapshotへ入れない。

MVP Secret参照はinternal Job `with` の**1 scalar全体**だけ許可する。

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

`"Bearer ${{ secrets.API_TOKEN }}"` のような埋め込みは禁止。実加工はAction側。

Result validation順:

1. JSON-compatible / canonical JSON
2. optional JSON Schema
3. optional Validator
4. optional `success_if`
5. SecretGuard
6. PayloadStore

Validatorはinternal/externalのみ。Human/Reusable parent Jobでは不可。

Known Secret:

- persistent Output/State/Artifact metadata/Event/error reject
- managed Artifact content scan
- Execution Log redact

## 8. Action / Validator Registry

YAMLはAction/Validator IDを指定し、versionはRun start時にRegistryから解決する。

MVP Registryは各Processで:

```text
action_id -> exactly one current version/callable
validator_id -> exactly one current version/callable
```

- ID/versionはnon-empty string
- 同じIDの二重登録reject
- Multi-version RegistryはMVP外
- Retry/Resume/Runner実行時はRun snapshot versionとcurrent Registry versionのexact一致を要求
- Version更新を忘れた場合は親責任
- Coreはsource codeを解析しない

Action sync/async。`ActionFailure(code,message,retryable,details)`対応。

Runtime Handle:

- progress / step / log
- state get/set
- cancel check
- managed Artifact put/materialize
- external Artifact reference registration

## 9. Runner Pool / Runner

Poolは親が事前登録。`runs-on`は登録済みPoolのみ。省略時はRun snapshotの`default_runner_pool`。

Pool configは少なくとも:

```text
name
runner_count >= 1
restart policy
heartbeat/lost/main-loop-stale settings
```

Poolごとに`runner_count`分を自動起動。

MVPでPoolに持たせないもの:

- Action allow/deny list
- CPU/RAM/GPU quota
- OS sandbox
- Pool global pause

Same Workflow Run internal Job同時実行最大1。別RunはRunner数まで並列。

Job selection:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source order ASC
5. ready time ASC
6. deterministic ID

## 10. Runner / IPC / Heartbeat

Runner管理ProcessとCommon Action Runner子Processを分離。

Default:

```text
heartbeat = 5s
runner_lost = 20s
main_loop_stale = 15s
```

SupervisorがOS process + heartbeatを監視。重いActionは別Processなのでheartbeatと分離。

Runner↔Action Runnerは専用JSON Lines v1。stdout/stderrはExecution Log。

Handshakeは`ready -> start -> result|error -> process exit`を正規順序とし、Runtime Handle request/responseは`request_id`で対応付ける。

Large Action resultはworkdir reserved `result.json`へ書き、IPCはpath/size/digestのみ。

Parent正常shutdownはWorkflow cancelではない。未完了running Attemptは次回`runner_lost` Recovery。

## 11. Job activation / readiness / Retry pending snapshot

`queued` JobはRunner取得可能とは限らない。

```text
ready_at = NULL     -> dependency/condition activation待ち
ready_at != NULL    -> 次Attempt用Input snapshot確定済み
```

初回internal Jobはactivation時にJob rowへpending Input snapshotを保存し、Runner claim時にAttemptへexact copyする。

External/Human/Reusable初回activationは直接Attemptを作る。

Automatic/Manual Retryでは**全executor共通**でfailed AttemptのInput/bindings/digestをJob pending snapshotへcopyし、次Attemptは実際の再実行開始時に作る。

Resume/Recoveryでready済みpending Inputを再評価しない。

## 12. Maintenance Loop / Timeout

内部deadline/housekeeping専用。User Workflow Schedulerではない。

対象:

- `retry_not_before`
- External Lease `expires_at`
- concurrency wake
- Retention
- orphan cleanup
- expired idempotency cleanup

Default max sleep5秒、busy loop無し。

`timeout-minutes`はinternal Jobのみ、hidden default無し。ExternalはLease、Human期限無し、ReusableはChild内Job timeout。

## 13. 状態 / 条件

Job status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Job conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Workflow Run status:

```text
queued|running|paused|completed
```

Workflow conclusion:

```text
success|failure|cancelled
```

`if` default=`success()`。

Effective success=success または failure+continue-on-error。Normal skipped dependencyは後段default skip。

Cancel後は`always()`でもnew activation無し。

## 14. JSON Output / PayloadStore

Action/External resultは任意JSON-compatible value。

Default inline threshold=4MiB。

- <= threshold: SQLite inline
- > threshold: filesystem PayloadStore

4MiBはmaxではない。Downstreamはstorage kindを意識しない。

OutputはAttempt単位immutable。Current Job Outputはcurrent successful Attemptから解決。

## 15. ArtifactStore / ArtifactRef

ArtifactはOutputとは別の明示成果物。

Managed:

- `runtime.artifact.put_file`
- temp自動Artifact化無し
- Retry same name=new generation
- Core Store管理/materialize可能

External reference:

- URI metadata
- Core fetch/materialize/delete無し

Canonical ArtifactRefはJSON objectでInput/Outputへ明示mappingできる。

Cross-run Managed Artifact利用:

- ArtifactRefがcurrent persistent Job Inputへ明示存在
- Artifact data存在
- AuthorizationProviderがsource Artifact readを許可

が必須。Raw artifact_idだけでは不可。

Cross-run ArtifactRefはRetention holdを作らず、source dataがRetention済みならmaterializeはfail-closed。

## 16. Dynamic Job

Dynamic templateは**仮想group**であり、template自身の`job_runs` row / Attempt / Retry / Progress実行単位は作らない。`job_runs`へ入るのはgenerated concrete Jobだけ。

Root=`foreach` scalar、Nested=`foreach.parent/items`。

- fixed depth limit無し
- generated max default1000/Run
- expansion all-or-nothing
- stable key推奨
- full logical key parent path込み
- Action/Validator/Runner Pool preflight

`order_by`:

- omitted / `source_order`
- list `{expr,direction:asc|desc}`
- stable source-order tie-break

Outcomeを区別:

- `if=false` -> skipped
- `foreach=[]` -> normal empty success

Nested parent 0件はparent group conclusionを伝播し、存在しないparent itemを`always()`で生成しない。

Expanded expansionはsource/key/item/orderを含むdigestを保存し、同一Run内でgenerated Job集合を勝手に差し替えない。

## 17. Reusable Workflow

`uses`でChild Workflow Run。

Relative referenceはcaller source directory基準、Workflow root escape禁止。Non-filesystem callerはregistered ID。

Parent Job first activationでChild bindingを固定:

- Child Definition snapshot/hash
- Child Action/Validator versions
- Child effective runtime settings
- Child effective Retention policy

Retryはsame bindingからnew Child Runを作る。親のSystem設定変更で同一Parent RunのChild条件を変えない。

Parent/Child state非共有。

Artifactは暗黙共有せずArtifactRefを`with`/Workflow Outputへ明示mapping。Child ArtifactをParent Job artifacts namespaceへ自動mirrorしない。

Child direct public pause/resume/cancel/retry/priority変更は禁止。

## 18. External LLM / Human

External:

- activation時Attempt + Task
- Lease既定60分 / expiry既定requeue
- Job lease override > Workflow settings > System config > default
- effective lease/expiryはAttempt/Task生成時に固定
- atomic claim/owner submit
- Maintenance Loop expiry処理
- optional Validator
- `claim_next`
- Lease heartbeat/renew/extend/transferはMVP無し

`task_submit(claim_next=true)` は、submit terminal化 + optional next claim + idempotency resultを**同一DB transaction**で確定する。Replay時に別Taskを追加claimしない。

Human:

- pending Review
- approve/rejectのみ
- no Lease/timeout/Validator
- first-wins
- Output null
- completed Review outcome rewrite無し

MVPではfailed/cancelled Jobのmanual success override、generic manual skipを提供しない。失敗はRetry、skip/failure許容はDefinitionで事前定義。

## 19. Retry / Recovery

Retry=new Attempt。ただしRetry要求/予約時点ではnew Attemptを作らず、failed Attemptの:

```text
persistent Input
Secret bindings
Input digest
Definition / Action / Validator identity
Dynamic iteration
Reusable binding
```

を次Attempt用pending snapshotとして固定する。

- retry absent = Automatic Retry無し / max1
- `retry:{}` = max2 / `failure.retryable` / zero backoff

Manual Retryはfailed Job + failed Attempt/Input Snapshot必須。Pre-Attempt failureは`retry_input_unavailable`でnew Run要求。

Completed/failure Runをmanual retryする場合same Run reopen + `run_attempt++`。Recoveryだけではcompleted Runをreopenしない。

## 20. Same-Run Result Reuse

Automatic reuseはsame Workflow Runのみ。Cross-run/global cache無し。

Manual Retry後のsuccessful descendantは、保存済みkey同士だけでなく、**現在のdependency contextから副作用なしで再評価**する。

最低限:

- current `if` がtrue
- current `with`からexpected Input/bindings/digestを再構築して一致
- direct upstream Artifact identities一致
- whole Definition hash一致
- executor/Action/Validator identity一致
- stored Output/Artifact/Payload integrity
- Schema/Validator/`success_if`再検証
- `reuse_eligible=true`

`state.get`やpersistent Input外Artifact materializeはineligible。

再利用不能なら同一Run内でchanged Inputとして勝手に再実行せず、`successful_job_result_not_reusable`でnew Workflow Runを要求する。

Dynamic Jobについてもexpanded expansionのsource/key/item/order digestを再評価し、変化していれば`dynamic_expansion_not_reusable`でnew Run要求。

## 21. Pause / Cancel

Pause:

- running internal継続
- new internal/External claim/Dynamic expansion停止
- existing submit/review/Child進行可
- ready済みpending Input snapshot保持
- Lease expiry/Retention housekeeping継続

Cancel:

- new activation禁止
- queued/waiting cancel
- Lease invalidation
- running internal graceful cancel
- Child propagation

Public force-killはMVP無し。

## 22. Workflow State / Log / Step / Progress

State=get/set、persistent、last-write-wins、revision/history。CAS/increment無し。

Execution Logは**全executor Attempt共通**でfile管理する。External/Human/ReusableもEngine lifecycle/diagnosticを同じLog subsystemへ記録する。

Event Log=append-only DB。Retention audit Eventは別扱いで無期限。

Step=観測単位のみ。独立Scheduling/Retry/timeout/Artifact ownership無し。

Progress:

- Job mode=`auto|explicit|none`
- Workflow mode=`auto|none`
- explicitはAction Runtime Handleの`progress`を保存
- autoはterminal/known executor stateからEngineが算出
- Workflow autoは実行Jobを集約し、Dynamic template自身は分母へ数えない
- Reusable Parent JobはChild Workflow progressを集約可能

Temp directoryはAttempt終了時削除。Sandboxではない。

## 23. Persistence / Idempotency / Retention

Standard SQLite。WAL/FK/busy timeout。FK NO ACTION、明示delete order。

MVP canonical tableは18個。

重要constraint:

- one running internal / Run
- Dynamic root/nested expansion unique
- Dynamic templateは`job_runs` row無し
- Child / Parent Attempt unique
- one Task / Attempt
- one active Lease / Task
- one Review / Attempt
- state current+history atomic
- pending Input snapshot / Attempt Input snapshot整合

Idempotency:

- scope=`namespace + resource + AccessScope + Actor/client principal`
- request hashはcanonical normalized requestから算出
- Default TTL24h
- expired key再利用可
- completed resultだけ永続化し`reserved` rowは作らない
- commit transaction内で同一keyを再確認
- Replayは初回result/HTTP statusを再生し副作用再実行無し

Retention exact keys:

```text
run-history-days
execution-logs-days
event-days
artifact-metadata-days
managed-artifact-data-days
```

null=unlimited。System default全null。Workflow field > System > unlimited。Run start時snapshot。

Non-terminal current dataを期限だけで削除しない。Run history expiryがowned component最終上限。External Artifact実体は削除しない。

Cross-run ArtifactRefはsource retentionを延長しない。

Retention audit EventはRun FK無し、通常event retention対象外、MVP無期限保持。

## 24. Authorization / Service / MCP / HTTP

Authentication親責任。全public read/write + cross-run Artifact readをAuthorizationProvider経由。Default AllowAll。

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
wf_event_list
wf_runner_info
```

存在しないMVP API:

```text
lease heartbeat/renew/extend/transfer
manual Job skip
manual success/conclusion override
completed Review rewrite
```

MCP=`<namespace>_wf_*`、collision reject。

HTTP prefix=`/api/jobrunner/v1`、exact contract=`11`。HTTP status=200/201/400/401/403/404/409/413/500、422無し。

Run infoにOutput/Log/Event本文を無制限埋め込みしない。Output readはJMESPath select可。Eventは`wf_event_list`でページング取得。MCP oversized resultはsilent truncate無し。

## 25. Scheduler / CLI / WebUI

User Workflow Scheduler/Cron/CLIはMVP無し。

WebUI画面構成のみ後続。Service/HTTP contractはCore設計で固定済み。

## 26. Source of Truth

実装時は `docs/detailed-design/01`〜`13` をSource of Truthとする。

基本設計と詳細設計が衝突する場合は、より新しく具体的な詳細設計を優先し、基本設計も同期修正する。
