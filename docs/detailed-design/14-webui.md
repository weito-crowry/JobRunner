# 14. WebUI 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `03`, `04`, `07`, `08`, `09`, `10`, `11`, `12`, `13`
- Visual reference: `docs/ui/mockups/jobrunner-ui-mock.html`

## 1. 目的

JobRunner WebUIは、既存アプリケーションへ組み込まれた1つのJobRunner Runtimeを人間が通常運用・監視・調査するためのWeb control panelである。

WebUIはJobRunner Coreの別実装ではない。Workflow/Run/Job/Attempt/Review/State/Artifact/Runner等の状態のSource of TruthはCore Service/Persistenceであり、Web Adapterは同じService layerを利用する。

MVPでWebUIが担う通常運用:

- Dashboard
- Workflow Definition browse / read-only YAML / DAG
- Workflow start
- Workflow Run list/detail
- Pause / Resume / Cancel / Priority update
- Job / Attempt / Step inspection
- Manual Retry
- Execution Log follow/read/download
- Input / Output inspection
- Managed Artifact metadata/read/download
- Human Review approve/reject
- Workflow State current/history read-only
- Event history
- Runner / Runner Pool monitoring
- Runner restart history
- Runtime configuration read-only

MVP非目標:

- GUI Workflow editor / Definition write
- Workflow State public mutation
- Generic Job success/skip/force-complete/override
- Human Review rewrite
- Runner kill/restart/scale/Pool pause
- External Taskを人間がclaim/submitするworker UI
- Runtime setting write
- JobRunner独自Authentication/Login/User DB
- 複数Runtime中央管理console
- Cron/CLI/notification機能

## 2. 設計原則

1. **RESTがcanonical state。** WebUI上の状態はHTTP Service APIのresponseをSource of Truthにする。
2. **SSEは変更通知だけ。** SSE payloadへInput/Output/State/Event payload/Log本文等のbusiness dataを載せない。
3. **Execution LogはAttempt単位。** Step選択でLogをfilterしない。
4. **AdapterはDB/Storeへ直接アクセスしない。** 必要なread APIはService layerへ追加する。
5. **Mutationは既存Core rulesを再利用する。** WebUI専用の裏口mutationを作らない。
6. **Authenticationは親責任。** Parentが認証済みrequestから`ActorContext`/`AccessScope`を解決する。
7. **AuthorizationはService側で再判定する。** UIでbuttonを隠すことはsecurity boundaryではない。
8. **1 mount = 1 Runtime。** WebUI内にRuntime selectorを持たない。
9. **主要resourceは独立URLを持つ。** Browser reload / bookmark / URL共有で同じresourceを復元できる。
10. **Core IDはopaque。** UI用連番をCore IDの代わりに導入しない。
11. **strict/no-coercionを維持する。** Browser form都合でCore input typeを暗黙変換しない。
12. **既存Run snapshot semanticsを表示側で変更しない。** `started_at=null`等をUI convenienceで補完しない。

## 3. 技術構成

### 3.1 Frontend

```text
React
TypeScript
Vite
React Router
TanStack Query
TanStack Table
TanStack Virtual
Radix UI
shadcn/ui系components
Tailwind CSS
React Flow
CodeMirror 6
```

Test:

```text
Vitest
React Testing Library
MSW
Playwright
```

Global application state store（Redux/Zustand等）はMVPでは導入しない。

State ownership:

```text
Server state      -> TanStack Query
URL state         -> React Router path/search
Temporary UI      -> React local state
```

### 3.2 Backend Adapter

`jobrunner[web]`はFastAPIベースのWeb Adapterを提供する。

```text
Parent Application
├─ Parent Authentication
├─ JobRunner Runtime / Service
└─ JobRunner FastAPI Router
   ├─ HTTP JSON API
   ├─ SSE change stream
   └─ React SPA static assets
```

JobRunnerがproduction用FastAPI application全体を所有することを必須にしない。ParentはJobRunner Router/Sub-applicationを既存Web applicationへmount/registerする。

Dev/test用hostを提供してもよいが、別管理必須のJobRunner専用production daemonは導入しない。

## 4. Package / build

Python source:

```text
jobrunner/
└─ adapters/
   └─ web/
      ├─ router.py
      ├─ dependencies.py
      ├─ models.py
      ├─ errors.py
      ├─ streaming.py
      ├─ sse.py
      ├─ static.py
      └─ static/
```

Frontend source:

```text
webui/
├─ src/
│  ├─ app/
│  ├─ api/
│  ├─ components/
│  ├─ features/
│  └─ routes/
├─ tests/
├─ package.json
├─ package-lock.json
├─ tsconfig.json
└─ vite.config.ts
```

Build:

```text
npm ci
npm test
npm run build
    -> jobrunner/adapters/web/static/
    -> Python wheel
```

Node/npmは開発・package build時のみ必要。`pip install jobrunner[web]`後のruntimeでNode/npmを要求しない。

Base `jobrunner` packageはFastAPI等のWeb optional dependencyを持たない。

Vite hash assetはlong-lived immutable cache可。`index.html`は`no-cache`とする。

## 5. Mount model

### 5.1 1 mount = 1 Runtime

1つのWebUI Router instanceは1つのJobRunner Runtime / Service instanceだけを参照する。

Default logical paths:

```text
UI  /jobrunner/
API /api/jobrunner/v1/
```

Parent outer prefixによるmountを許可する。

例:

```text
/novel/jobrunner/
/novel/api/jobrunner/v1/

/fx/jobrunner/
/fx/api/jobrunner/v1/
```

同じ静的bundleを異なるbase pathへ再利用できること。

SPAは`/jobrunner`をcompile-time固定しない。Serverが返すHTMLの`<base href>`およびAPI base metadataからRouter/API baseを解決する。

### 5.2 Parent process制約

MVPでは、1つのmounted Runtime / Runner Pool Supervisorを所有するParent application processは1つとする。

同じRuntime/SQLite/Runner Poolを各processが独立bootstrapするような:

```text
uvicorn --workers N
```

構成はMVP対象外。

Reverse proxy等で1つのParent processをfrontする構成は可。

## 6. Authentication / Authorization

### 6.1 Authentication

JobRunner自身はLogin page、user/session DB、password/token発行を持たない。

Parent authentication middlewareが認証済みrequestから:

```text
ActorContext
AccessScope
```

を解決し、Web AdapterからCore Serviceへ渡す。

`ActorContext`/`AccessScope`へsession token/password等のcredentialを格納しない。

### 6.2 Authorization

WebUIのbusiness read/writeは既存`12`のAuthorizationProviderを必ず通す。

UI上の`available_actions`やbutton visibilityはUX projectionでありauthorization bypassには使えない。Mutation時にServiceがcurrent state/authorizationを再検証する。

SSE/bootstrapはAdapter control endpointでありbusiness resource本文を返さないが、Parent authentication成功を必須とする。Business resource取得は各REST Service operationで個別authorizeする。

### 6.3 Same-origin / CSRF

WebUIは原則Parent applicationとsame-originで提供する。JobRunnerがdefaultで広いCORSを有効にしない。

Cookie authenticationを使うParentではParentのCSRF protectionをJobRunner state-changing routeにも適用する。JobRunner routeを一括CSRF除外しない。

## 7. Global Navigation / URL

Top-level Navigation:

```text
Dashboard
Workflows
Runs
Reviews
Runners
Runtime
```

Job/Attempt/State/Event/Artifact/External Taskは関連resourceから辿る。

Canonical SPA routes:

```text
/jobrunner/
/jobrunner/workflows
/jobrunner/workflows/:workflowId
/jobrunner/workflows/:workflowId/start
/jobrunner/runs
/jobrunner/runs/:runId
/jobrunner/runs/:runId/state
/jobrunner/runs/:runId/events
/jobrunner/runs/:runId/jobs/:jobId
/jobrunner/runs/:runId/jobs/:jobId/attempts/:attemptId
/jobrunner/reviews
/jobrunner/reviews/:reviewId
/jobrunner/artifacts/:artifactId
/jobrunner/runners
/jobrunner/runtime
```

SPA baseが変更される場合もroute suffix semanticsは同じ。

ID displayは省略表示可だが内部route/queryではfull opaque IDを保持する。

## 8. Dashboard

目的は現在の実行状況・human intervention・Runner issueを短時間で把握し、関連list/detailへ遷移すること。

表示:

- Running Runs
- Queued Runs
- Paused Runs
- 指定期間内にcompleted/failureとなったRuns
- Pending Human Reviews
- Runner busy/idle/lost
- Restart suppressed
- Recent Runs
- Needs Attention

Dashboardはgeneric mutationを持たず、各resourceへのnavigationを主目的とする。

`completed failure`の期間表示を行う場合、Browser local dayをUTC rangeへ変換しcanonical timestampとしてrequestする。Service側でBrowser timezoneを推測しない。

## 9. Workflow screens

### 9.1 Workflow list

表示:

- workflow_id / name
- version
- description summary
- Job template count
- latest accessible Run summary

Definition sourceのcurrent invalid/unavailable状態をold cacheで正常表示しない。

### 9.2 Workflow detail

Tabs:

```text
Overview
Jobs
Definition
Runs
```

Overview:

- ID / name / version / description
- definition hash
- source kind/display
- Input schema summary
- Output definition
- Job count

Jobs:

- read-only DAG
- Job template list
- DAGはlogical dependency viewでありscheduler execution orderの保証表示ではない

Definition:

- read-only source YAML
- GUI editor/save control無し

Runs:

- current Workflow IDでfilterしたRun history

## 10. Workflow Start

独立route `/workflows/:workflowId/start`を使用する。

Input editorは2 mode:

```text
Form
JSON
```

### 10.1 Form mode

JSON Schemaからdeterministicにcontrolを生成できるprimitive/object/array/enum/required/defaultをform化する。

複雑なschema構造をUIが安全にform化できない場合、そのsubtreeまたは全体をJSON editorへfallbackする。JSON editorはvalid Workflow Input objectを表現できるcanonical fallbackである。

FrontendがSchema semanticsを狭めることでvalid Core inputを拒否してはならない。

### 10.2 Type rules

Formはtyped valueを生成する。

```text
"12" != 12
"true" != true
true != 1
```

UI独自のcoercion/default補完は禁止。

明示されたschema/Workflow defaultだけを初期値として扱う。JSON modeからFormへ戻す際にinvalid値を勝手に修正しない。

Secret materialized valueを入力する専用fieldは提供しない。Secretは既存Core binding semanticsに従う。

### 10.3 Start

State-changing HTTPは既存通り`Idempotency-Key`を使う。

Submit成功後:

```text
/runs/:runId
```

へ遷移する。

Concurrency waiterで`started_at=null`の場合もそのまま表示する。

## 11. Run list

Filters:

```text
workflow_id
status
conclusion
created_from
created_to
```

filterはURL search paramsにも保存しreload/share可能にする。

`status`と`conclusion`を別fieldとして表示する。

例:

```text
status=completed
conclusion=failure
```

を独自のRun statusへ変換しない。

## 12. Run detail

WebUIの中心画面。GitHub Actions風にJobsを見渡しつつ、Run全体のinspection/controlへ遷移できる構成にする。

表示:

- workflow identity/version
- full/abbreviated Run ID
- status/conclusion
- progress
- priority
- run_attempt
- created_at / started_at / completed_at
- wait_reason / concurrency_queued_at
- parent/root Run navigation
- Jobs / Dynamic groups
- Input / Output
- State
- Events

### 12.1 Run controls

Core Serviceの`available_actions` projectionを表示判断に使う。

対象:

```text
pause
resume
cancel
priority_update
```

Child Runはdirect controlを表示せず、Parent controlledであることを表示する。

Pause/ResumeのConcurrency semanticsをUIで変えない。

- admitted holder Pauseはslot保持
- concurrency waiter Pauseはslot無し
- paused waiterは元`concurrency_queued_at`保持

Cancelはconfirm dialog + optional reason。Priorityはsigned64をそのまま扱いUI独自rangeへ狭めない。

## 13. Job detail

独立URLを持ち、permanent drawerだけにはしない。

表示:

- job_run_id / job_key / template key
- executor
- status/conclusion
- priority/progress
- failure
- current Attempt
- Attempt history
- Input / Output
- Artifacts
- External Task / Human Review / Child Run navigation metadata

Generic:

```text
Mark success
Skip
Force complete
Override conclusion
```

は提供しない。

Manual Retryは`available_actions.retry=true`の場合だけ表示し、Service mutation時に再検証する。

Dynamic template virtual groupはConcrete Jobとして扱わずAttempt/Logリンクを持たない。Generated Concrete Jobのみ通常Jobとして表示する。

## 14. Attempt / Step

Attempt detailは独立URLを持つ。

Tabs:

```text
Overview
Steps
Log
Input
Output
Artifacts
```

Attempt numberはhistory上のordinalとして表示可能だがopaque `attempt_id`を置換しない。

StepはAction runtime observationである。

禁止:

- StepへJob schedulingのqueued/waiting/skippedを流用
- 定義から未作成future Stepを先読み表示
- Step nesting/parallelをMVP UI上で捏造

DBに存在するStepだけを表示する。

## 15. Execution Log

Execution LogはAttempt-levelで最大1 logical log/Attempt。

Step selectionはLog filter条件にならない。

UI:

- Tail read
- Follow
- Line wrap
- Client-side visible-text search
- Full log download

初回はtailを取得し、Running Attemptのfollowは既存`wf_log_read`の`next_offset_bytes`を使ったoffset pollingを行う。

Frontendが文字数からbyte offsetを計算しない。常にBackend responseの`next_offset_bytes`を次requestへ渡す。

Browserが末尾からscroll awayした場合はauto-followを停止し、新line count/末尾移動controlを表示する。

RetentionでLog dataが削除済みの場合はempty logとして表示せず`log_data_unavailable`を表示する。

## 16. Input / Output

Run/Job/Attemptそれぞれ既存info/read APIを利用する。

初期画面で本文を一括取得せず、infoからavailability/size/digest等を確認し、tab open時に本文を取得する。

Viewer:

```text
Tree
Raw JSON
```

JMESPath `select`は既存Service semanticsを利用する。

Input上のSecretはpersistent referenceだけを表示し、materialized Secret valueを取得・表示しない。

## 17. Workflow State

Run単位のState current/historyをread-onlyで表示する。

Listはmetadataのみ。Valueは項目選択時に`wf_state_read`で取得する。

Historyはまず`include_values=false`で取得し、必要時だけ値を読む。

WebUIからState write/deleteを提供しない。

Attempt failure後にもState historyが残り得る既存semanticsをそのまま表示する。

## 18. Event timeline

Run/Job/Attempt chain filterで既存Event APIを使用する。

Event type raw identifierを失わず表示する。

例:

```text
job_completed
Conclusion: success
```

Payloadは折りたたみJSON。Eventは監査/履歴でありSSE notificationとは別概念。

## 19. Human Review

Top-level ReviewsはHuman operatorの作業queue。

Default list filterはcanonical Review status `pending`。

Review detail:

- review_id
- Run / Job / Attempt navigation
- Input
- created_at
- current status/outcome
- comment

Mutation:

```text
approve
reject
```

のみ。Outcome rewrite、Output rewrite、skip等を追加しない。

Submitはconfirmし、state raceで409となった場合は最新Reviewを再取得してcompleted/cancelled状態へ更新する。

## 20. Artifact

ArtifactはProducer Run/Job/Attemptから辿る。MVPでtop-level Artifact listは必須にしない。

Detail:

- Artifact ID
- name
- storage kind
- media type
- size
- digest
- URI（Externalのみ）
- producer
- metadata / deletion state

### 20.1 Managed Artifact

Authorized inspection用にManaged data read/downloadを追加する。

Public inspectionのartifact ID selectorはdataflow dependency作成ではない。Actionによるcross-run materializeのArtifactRef要件を緩和しない。

ServiceがArtifact row/data existence、storage kind、Authorization、Store existence/integrityを確認してstream handleを返し、HTTP AdapterはStore pathを直接組み立てない。

### 20.2 External Reference

JobRunner Coreは外部URIをproxy fetchしない。

WebUIはURI/metadataを表示するがManaged Download buttonは表示しない。

## 21. Runner / Runtime

### 21.1 Runners

Poolごとに:

- configured count
- Runner logical/instance identity
- status
- current Job metadata（存在する場合）
- heartbeat/liveness
- restart status/history

を表示する。

MVPでRestart/Kill/Scale/Pool Pause操作は提供しない。

### 21.2 Runtime

Current Runtime/configurationをread-onlyで表示する。

`wf_runtime_info`は既存canonical configuration modelに存在する設定値だけを返し、新しいhidden defaultを作らない。

Run lineageへsnapshot済みのeffective settingsとcurrent Runtime settingsを混同しない。

Runtime write APIはMVPに追加しない。

## 22. Loading / Empty / Error

### 22.1 Loading

Initial loadはSkeleton。Background refetch時は既存stale dataを消さない。

### 22.2 Empty

合法的な0件をerrorと区別する。

例:

```text
No workflow runs yet.
No reviews are pending.
This workflow run has no state entries.
No log output yet.
```

Retentionで削除されたdataはEmptyではなくUnavailable error。

### 22.3 Error

3層:

```text
Page-level
Section-level
Mutation-level
```

Canonical Service error `code/message/retryable/field-or-path/details`を保持し、HTTP statusだけのgeneric errorへ潰さない。

Mutation 409後は対象resourceを再取得する。

## 23. Mutation UX / Idempotency

State-changing HTTPは既存`Idempotency-Key`を使用する。

1 user intentにつき1 keyを生成し、通信結果不明のretryでは同じkeyを再利用する。Userが内容を変更して新しいintentを送る場合は新key。

強いoptimistic domain updateはしない。

```text
request
-> Service success
-> response反映
-> query invalidate/refetch
```

とする。

Cancel/Retry/Reviewはconfirmationを要求する。Pause/Resumeは通常dialog無し。Mutation中は対象actionだけdisableしページ全体をlockしない。

## 24. `available_actions`

FrontendへCore state machineを複製しないため、resource detail responseへcaller-specific read projectionを追加する。

Run:

```json
{
  "available_actions": {
    "pause": true,
    "resume": false,
    "cancel": true,
    "priority_update": true
  }
}
```

Job:

```json
{
  "available_actions": {
    "retry": false
  }
}
```

Review:

```json
{
  "available_actions": {
    "submit": true
  }
}
```

Projectionはcurrent domain state + Authorizationを評価したsnapshotでありlock/reservationではない。Mutation時のraceにより後続requestが409となることを許容する。

Authorizationにより利用不可な操作はUIで表示しない。Child Run等のdomain structure上の制約は必要に応じ説明textを表示する。

## 25. Web bootstrap

Adapter endpoint:

```text
GET /api/jobrunner/v1/web/bootstrap
```

ResponseはWeb integration metadataだけを返す。

例:

```json
{
  "runtime_instance_id": "runtime_...",
  "system_namespace": "novel",
  "display_name": "NovelProduction",
  "api_version": "v1",
  "webui_version": "1.0.0",
  "features": {
    "workflow_start": true,
    "run_control": true,
    "review": true,
    "artifact_data_read": true,
    "sse": true
  }
}
```

`features`はAdapter build/mount capabilityでありresource authorizationではない。

credential、Secret、session tokenを返さない。

## 26. WebUI用Service/API拡張

WebUI要求により以下のread contractを追加する。Web AdapterからDB/Storeへ直接読みに行く代替は禁止。

### 26.1 `wf_dashboard_info`

Dashboard aggregate projection。

Request:

```text
completed_from canonical timestamp optional
recent_limit 1..50 default10
```

ResponseはcallerのAccessScope/Authorizationで閲覧可能なresourceだけを集計する。

```text
run_counts: running/queued/paused
completed_failure_count
pending_review_count
runner_counts: busy/idle/lost
restart_suppressed_count
recent_runs[]
```

### 26.2 Definition list projection

`wf_definition_list` itemへ後方互換追加:

```text
job_template_count
latest_run_summary nullable
```

`latest_run_summary`はcallerがread可能なRunだけを対象にする。

### 26.3 `wf_job_info`

Request:

```text
job_run_id
include_attempts=true
include_steps=false
include_artifacts=true
```

既存`wf_run_info`内のcanonical Job/Attempt/Step shapeを再利用する。別意味のduplicate modelを作らない。

### 26.4 `wf_attempt_info`

Request=`attempt_id`。

Response:

```text
attempt_id/attempt_no
workflow_run_id/job_run_id
status/conclusion
input_digest
output_metadata nullable
failure nullable
started_at/completed_at
log_metadata
external_task_id nullable
review_id nullable
child_workflow_run_id nullable
steps[]
artifacts[]
```

### 26.5 `wf_artifact_data_read`

Managed Artifact inspection dataをstream可能なhandleとして返すService operation。

Authorization=`artifact.read`。

External Referenceは`artifact_data_unavailable`相当でdata streamを提供しない。

Store path/credentialはpublic responseへ出さない。

### 26.6 `wf_log_stream`

Execution Log全体をdownload/streamするService operation。

既存`wf_log_read`と同じAttempt ownership/Authorization/Retention rulesを使う。Deleted Logは`log_data_unavailable`。

StreamingはHTTP/Python Adapter用途で、MCPに巨大Log本文を埋め込むことを必須にしない。

### 26.7 `wf_runner_restart_list`

Filters:

```text
pool optional
runner_id optional
created_from optional
created_to optional
limit/cursor
```

Restart decision historyをread-onlyで返す。`08`の`runner_restarts` semanticsを変更しない。

### 26.8 `wf_runtime_info`

Current Runtime identity + validated current configurationのread-only projection。

Run snapshot済みeffective settingsをcurrent設定で上書きしない。

### 26.9 MCP exposure

上記Service operationの存在とMCP tool exposureは別概念。

Parent integrationはMCP adapterで公開するtoolを選択可能とし、WebUI用read operationを全てMCPへ自動公開することを必須にしない。

## 27. Web HTTP routes

Existing `11` routesに加え、WebUI用に:

```text
GET /dashboard
GET /jobs/{job_run_id}
GET /attempts/{attempt_id}
GET /artifacts/{artifact_id}/data
GET /attempts/{attempt_id}/log/data
GET /runner-restarts
GET /runtime
GET /web/bootstrap
GET /stream
```

を追加する。

`/artifacts/{id}/data`と`/attempts/{id}/log/data`はstreaming response。

HTTP error/status/idempotency policyは`11`を継承する。

## 28. Frontend data fetching

TanStack Query keyはresource identityに対応させる。

例:

```text
["dashboard"]
["workflow-definitions", filters]
["workflow-definition", workflowRef]
["runs", filters]
["run", runId]
["job", jobId]
["attempt", attemptId]
["input", type, id]
["output", type, id]
["state-list", runId]
["state", runId, name]
["state-history", runId, name]
["events", filters]
["reviews", filters]
["review", reviewId]
["artifact", artifactId]
["runners", pool]
["runner-restarts", filters]
["runtime"]
```

画面遷移/filter変更で不要requestをAbort可能にする。

API TypeScript型を手書き複製せずFastAPI OpenAPIから生成し、CIでgenerated type driftを検査する。

## 29. SSE / change feed

### 29.1 目的

SSEはBrowser cache invalidationのlatencyを下げるための非durable change notificationであり、Persistent Event Logの代替ではない。

### 29.2 Cross-process制約

Runner ProcessはParentとは別Processで同じSQLiteへcommitする。このためMVP WebUIのためだけにCore process間broker/pub-subを新設しない。

既存18-table persistence契約へWeb notification tableを追加しない。

### 29.3 `WebChangeFeedService`

Parent Runtime側にread-only change detectorを置く。

専用のlong-lived SQLite read connectionで`PRAGMA data_version`等のDB change signalを監視し、他connection/process commitを検出する。

Web Adapter自身がRepository tableを直接queryしてdomain dataを作らない。

短時間の複数commitは250〜500ms程度のcoalescing windowで1つのnotificationへまとめてよい。Exact windowは性能testで固定し、business semanticsには使わない。

### 29.4 SSE endpoint

```text
GET /api/jobrunner/v1/stream
Accept: text/event-stream
```

MVP notification:

```text
event: jobrunner
data: {"type":"stream.ready", ...}

event: jobrunner
data: {"type":"runtime.changed", ...}
```

`runtime.changed`へresource ID/business payloadを含めない。

KeepaliveはSSE commentで送信可能。

### 29.5 Reconnect

SSE notificationはdurable replay logではない。Replay buffer/Last-Event-IDによる完全再送をMVP必須にしない。

再接続成功時`stream.ready`を送り、Frontendは現在activeなqueryをrefetchする。

通知欠落によってstateを失わない。REST stateがcanonicalだからである。

### 29.6 Polling fallback

SSE接続失敗・連続再接続失敗時はREST pollingへfallbackする。

目安:

```text
Running Run detail    3s
Attempt Log           1-2s
Reviews               10s
Dashboard             10s
Runners               10s
Completed Run         auto polling停止
Definition            auto polling不要
```

Browser background時は頻度を下げてよい。SSE復帰時は重複pollを停止する。

## 30. SSEとPersistent Eventの分離

Persistent Event:

- append-only audit/history
- `workflow_started`, `job_completed`, `runner_lost`, `human_review_submitted`等
- public `wf_event_list`で読む

SSE:

- non-durable cache invalidation
- `stream.ready`, `runtime.changed`
- auditとして保存しない

「SSE 1件 = Event table 1件」という対応関係を要求しない。

## 31. Static / SPA serving

Routing order:

```text
API route              -> API
existing static asset  -> asset
SPA route              -> index.html
missing asset          -> 404
```

不存在`.js/.css`へSPA indexを200で返さない。

Hash asset:

```text
Cache-Control: public, max-age=31536000, immutable
```

`index.html`:

```text
Cache-Control: no-cache
```

## 32. Security / browser content

以下はuntrusted text/JSONとしてrenderする。

- Workflow description/YAML
- Execution Log
- Failure message/details
- Event payload
- Review Input/comment
- Artifact metadata
- State value
- Input/Output

`dangerouslySetInnerHTML`等で任意HTMLとして表示しない。

MVPでMarkdown rendererをbusiness textへ自動適用しない。

Artifact filename/Content-Dispositionはheader injection/path injectionを防ぐようsanitizeし、internal Store pathを公開しない。

WebUI local/session storageへInput/Output/Log/Secret/business dataを永続cacheしない。UI preferenceを保存する場合もbusiness payloadとは分離する。

## 33. Status / time / identity表示

StatusとConclusionを別概念として扱う。

Workflow Run:

```text
queued|running|paused|completed
```

Job:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

`skipped`をeffective successとして見せない。

日時はAPI canonical UTCをBrowser local timezoneへ表示可能。Relative timeだけにせずabsolute canonical timestampを確認できるUIを持つ。

opaque IDは短縮表示+copy/full tooltip可。短縮文字列をrequest identityに使わない。

## 34. Accessibility / responsive

Desktop-first。Tablet/narrow widthでnavigation/sidebarをdrawer/selectへ縮退可能とする。Mobile専用別UXはMVP非目標。

必須:

- Keyboard navigation
- Visible focus
- Dialog focus trap / Escape
- Statusを色だけで識別しない
- icon-only button accessible label
- field error association
- Log/JSON viewer accessible label

## 35. Performance baseline

以下でもBrowser main threadを不必要にblockしないこと:

- default上限近い1,000 generated Jobs
- multi-MiB JSON Output
- tens-of-MiB Execution Log
- long Event history

Large list/logはvirtualization/pagination/streamingを使用し、全量DOM renderを避ける。

## 36. Test strategy

### 36.1 Frontend Unit

- status/conclusion formatting
- timestamp/duration
- query parameter encode/decode
- ID abbreviation
- canonical API error mapping
- Workflow Input typed form conversion
- JSON/Form switching
- SSE connection/fallback state
- Log next byte-offset handling

### 36.2 Component

主要componentの:

```text
Loading
Success
Empty
Forbidden
Not found
Conflict
Unavailable retained metadata/data
```

をtestする。

### 36.3 FastAPI integration

Temporary real SQLite + Serviceで:

- Definition/list/info
- Start
- Run/Job/Attempt reads
- Pause/Resume/Cancel/Priority/Retry
- Human Review
- State/Event
- Artifact streaming
- Log streaming
- Dashboard
- Runner restart history
- Runtime info

を確認する。

AdapterがRepository/Storeへdirect business mutationしないことを構造testで固定する。

### 36.4 Authorization negative

UI button有無に関係なくHTTP direct accessで:

- read forbidden
- mutation forbidden
- state forbidden
- artifact forbidden
- review forbidden

がside effect無しで拒否されること。

### 36.5 SSE

- connect / keepalive
- dedicated connection change detection
- Runner ProcessからのSQLite commit検出
- coalescing
- disconnect / reconnect
- `stream.ready`
- polling fallback

### 36.6 Browser E2E

通常:

```text
Workflow list
-> Definition
-> Start
-> Run
-> Job
-> Attempt
-> Log follow
-> completion
-> Output
-> Artifact
```

Human:

```text
waiting_review Job
-> pending Review
-> approve/reject
-> Run continuation/terminal
```

Failure:

```text
failure
-> retry available
-> retry
-> new Attempt
```

Direct reload:

```text
/workflows/:id
/runs/:id
/runs/:id/jobs/:id
/runs/:id/jobs/:id/attempts/:id
/reviews/:id
/artifacts/:id
```

を一覧画面経由無しで復元できること。

### 36.7 Strict Input

最低限:

```text
"12" != 12
"true" != true
true != 1
required/null/default
array/object
```

をBrowser->HTTP->Coreまでtestする。

### 36.8 Accessibility

axe等の自動検査に加えてkeyboard/focus/dialog/form errorの主要flowをE2E確認する。

## 37. MVP受入条件

1. React + TypeScript SPAが`jobrunner[web]` wheelへbuild済みassetとして同梱される。
2. Node/npm無しでinstalled WebUIをserveできる。
3. Parent AuthenticationからActorContext/AccessScopeを注入できる。
4. 全business read/writeがService Authorizationを通る。
5. 1 mountが1 Runtimeだけを見る。
6. 主要resource URLはreload/bookmark可能。
7. Workflow Definition/YAML/DAGはread-only。
8. Workflow StartはSchema Form + JSON fallbackを持ちstrict typeを維持する。
9. Run status/conclusion/timestamps/concurrency semanticsをCore通り表示する。
10. Root non-terminal RunのPause/Resume/Cancel/Priorityが`available_actions`に従って操作できる。
11. Child Run direct controlsを提供しない。
12. Job detailでAttempt history/Failure/Input/Output/Artifactを確認できる。
13. Retryはeligible Jobだけに表示し、existing `wf_retry` semanticsを使う。
14. Stepはruntime observationとしてのみ表示する。
15. Execution LogはAttempt-levelで、Step filterを実装しない。
16. Running Attempt LogをBackend `next_offset_bytes`でfollowできる。
17. Managed Artifactをauthorized stream downloadできる。
18. External ArtifactをCoreがproxy fetchしない。
19. State current/historyをread-onlyで確認できる。
20. Persistent Event historyをresource chainで確認できる。
21. pending Human Reviewをapprove/rejectできる。
22. Runner/Pool/Restart historyをread-onlyで確認できる。
23. Runtime configurationをread-onlyで確認できる。
24. Runtime/Runner management write controlを持たない。
25. SSE `runtime.changed`でactive query refreshを促進できる。
26. SSE断でもREST pollingへfallbackしstate consistencyを失わない。
27. Runner Process commitがSSE change detectionへ反映される。
28. Secret materialized valueをWebUI/APIへ露出しない。
29. Major mutation raceはcanonical 409を表示してresource refetchする。
30. Direct URL reload、Authorization negative、strict input、Human Review、Retry、Artifact/Log streamingのE2Eが通る。

## 38. Visual mockの位置付け

`docs/ui/mockups/jobrunner-ui-mock.html`はVisual Direction Referenceでありnormative runtime/API contractではない。

優先順位:

```text
Core detailed design (`01`-`13`)
-> 本書 `14-webui.md` のWebUI-specific extension
-> UI mock
```

本書が既存Core semanticsを変更すると解釈してはならない。本書で追加するread projection/streaming/Web notificationはCore invariantsを維持する。

## 39. 関連文書へ同期すべきnormative delta

本書承認後、Source of Truth整合のため少なくとも以下を同期する。

### `docs/design.md`

- `WebUI: 画面構成のみ後続`をMVP WebUI詳細設計済みへ更新
- React SPA + FastAPI Adapter + parent embedding + read-only boundariesを要約

### `11-service-api-and-mcp.md`

- Service components/read operations: dashboard/job info/attempt info/artifact data/log stream/runner restart/runtime
- `available_actions`
- HTTP追加routes
- MCP exposureはparent-selectableであること

### `12-security-and-secrets.md`

- Web AdapterもParent Authenticationを利用
- Artifact data/Log stream Authorization
- WebUIがSecret materialized valueを取得しないこと

### `13-testing.md`

- FastAPI Adapter/React/SSE/streaming/E2E/strict form/Authorization negative test

同期時に本書と既存Core semanticsが衝突する場合、Core semanticsを無言で変更せず、設計差分として明示的に再レビューする。

## 40. Implementation phase分割

実装計画では概ね以下へ分ける。

### Phase 1 — Web foundation

- FastAPI Router / SPA serving
- Parent auth hooks
- OpenAPI -> TypeScript generation
- React shell/routing/bootstrap

### Phase 2 — Read-only operations

- Dashboard
- Workflows
- Runs
- Job / Attempt
- Input / Output
- State / Events
- Runners / Runtime

### Phase 3 — Mutations

- Workflow Start
- Pause / Resume / Cancel
- Priority
- Retry
- Human Review

### Phase 4 — Streaming

- SSE / Polling fallback
- Execution Log follow/download
- Artifact download

### Phase 5 — Hardening

- Authorization negative
- Cross-process SSE
- large data/performance
- accessibility/responsive
- packaging/E2E
