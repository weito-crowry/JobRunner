# JobRunner 基本設計

- Status: Draft v0.7
- 対象: MVP
- WebUI: 画面設計のみ後続
- 用語: GitHub Actions に対応概念がある場合は可能な限り合わせる

## 1. 目的

JobRunner は既存アプリケーションへ組み込んで使う軽量な Persistent Workflow / Job Runtime。

提供:

- Workflow / Workflow Run
- Job / Attempt / Step
- `needs`
- Action Registry
- Runner / Runner Pool
- Retry / Resume / Pause / Cancel
- Run History
- JSON Outputの透過永続化
- ArtifactStore / Artifact参照
- Human Review
- External LLM pull execution
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log
- SQLite persistence
- MCP / Web / Python 共通Service

非目標: Airflow/Temporal/n8n級の独立大規模基盤、Kubernetes/distributed scheduler、任意shell標準機能、GUI workflow editor、CLI/Cron、通知、完全GitHub Actions互換、本格sandbox、認証基盤。

## 2. MVP Python / OSS

Python >=3.10。

```text
ruamel.yaml >=0.19,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

SQLite/process/JSON/UUID等はPython標準libraryを優先する。

## 3. 用語

| 用語 | 意味 |
| --- | --- |
| Workflow | YAMLの設計図 |
| Workflow Run | Workflowを1回起動した実行実体 |
| Job | Workflow内の処理単位 |
| Attempt | Retryを含むJobの各試行 |
| Step | Job内の観測/ログ/進捗単位 |
| Action | 親がRegistryに登録する実処理 |
| Runner | internal Jobを実行する常駐Process |
| Runner Pool | Runner group |
| Artifact | Actionが明示的に登録するimmutable成果物 |

正式用語は`Runner`。`Worker`は使わない。

## 4. 組み込み構成

```text
Parent System
├─ JobRunner Runtime / Service
├─ Action Registry
├─ SQLite
├─ PayloadStore
├─ ArtifactStore
├─ MCP Adapter optional
├─ Web Adapter optional
└─ Runner Pools
   └─ Runner Process
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
```

親起動時にRuntime/Runner Poolを自動初期化。JobRunner専用サービスの手動起動は不要。

1 Repository / 1 Python Package。MCP/Webはoptional dependency。

## 5. Workflow Definition

Canonical formatはYAML。safe loader、duplicate key/merge key/custom tag/unknown keyをreject。

`env`はJSON-compatible literal-only。Expression/Secret参照は禁止。

Run開始時に実使用定義をsnapshot:

- Workflow ID/version
- source YAML
- canonical definition JSON/hash
- Input snapshot
- Action ID/version
- optional `source_identity`

既存Runは元YAML変更の影響を受けない。

## 6. Expression / Secret

GitHub Actions風`${{ ... }}`。

- CEL: condition/value計算
- JMESPath: JSON filter/projection

Secretsはinternal Action Jobの`with`だけ。`env`、External/Human/Reusable、condition等では禁止。

Current Attemptでmaterializeしたknown SecretはSecretGuardで:

- Output/State/Artifact metadata/Event/errorへの永続化をreject
- managed Artifact fileのexact byte sequenceも保存前scan
- Log/stdout/stderrは`[REDACTED]`

変形Secretの完全検出は保証しない。

## 7. Action Registry

親bootstrapでProcess起動時に再構築。CallableをDB/pickle転送しない。

Actionは`action_id + version`。sync/async対応。

Runtime Handle:

- progress / step / log
- state get/set
- cancel check
- managed Artifact put/materialize
- external Artifact reference登録

## 8. Runner / Runner Pool

Poolは親が事前登録。未登録はRun開始前error。

`runs-on`省略時System `default_runner_pool`、既定文字列`default`。

Runnerは常駐しJobをpull。

**同一Workflow Runのinternal Jobは同時に最大1件。** 別Runは並列可能。

選択順:

1. Workflow Run priority
2. Job priority
3. Dynamic order rank
4. source order
5. ready time
6. deterministic ID

External task claimも同じordering軸。

## 9. Runner / Action Process / IPC

Runner管理ProcessとAction子Processを分離。HeartbeatはRunner内別Thread。

Default:

- heartbeat 5秒
- Runner lost 20秒

Runner↔Action Runnerは専用JSON Lines channel。stdout/stderrはExecution Logへ分離。

Action Returnが巨大でもJSON Linesへ本体を載せない。Common Action RunnerがAttempt work_dirのreserved `result.json`へcanonical JSONを書き、IPCではpath/size/digestの小さい参照だけ返す。Runner検証後PayloadStoreへ保存する。

Runtime Handle request/responseもrequest_id付きIPCで処理する。

Parent正常shutdownはWorkflow cancelではない。旧Runtimeの未完了running Attemptは次回起動時に`runner_lost` Recoveryへ流す。

## 10. Timeout

Hidden default timeout無し。

`timeout-minutes`はinternal Jobだけ。

External LLMはLease、Human Reviewは期限なし、ReusableはChild内Job timeoutで制御。

## 11. 状態 / 条件

Job status:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Workflow Run:

```text
queued|running|paused|completed
```

Conclusion:

```text
success|failure|cancelled
```

`if`未指定は`success()`。

Effective success:

- success
- failure + `continue-on-error=true`

通常Jobの`skipped`はeffective successではなく、後段は明示条件が無ければskip。

Dynamic template groupは個別skipをgroup集約した結果としてsuccessになり得る。

Workflow cancel後は`always()`でもnew activationしない。

## 12. JSON Output / PayloadStore

Action/External Job resultは任意のJSON-compatible value:

```text
null / boolean / number / string / array / object
```

Optional JSON Schema / `success_if`を利用可能。

Workflowトップレベル`outputs`は名前付きmappingなのでWorkflow Output全体はJSON object。各field値は任意JSON-compatible value。

Default `output-inline-threshold-bytes = 4 MiB`。

- threshold以下: SQLite inline
- threshold超: durable filesystem PayloadStoreへ自動spill

**4MiBは上限ではない。大きいOutputも同じJSONとして成功・参照可能。**

Downstream `needs.*.outputs` / Workflow Output / Serviceは保存場所を意識しない。

Blobはsize/digestでintegrity検証。Missing/corruptionはfail-closed。

## 13. ArtifactStore

ArtifactはOutputとは別で、Actionが明示的に残すimmutable成果物。

### Managed Artifact

`runtime.artifact.put_file(...)` でAttempt work_dir内のfileをArtifactStoreへdurable copy。

- Temp fileを自動Artifact化しない
- same name retryはnew generation
- downstreamはArtifactRefで参照
- managed ArtifactはRuntime Handleからcurrent work_dirへmaterialize可能
- retention時はArtifactStore実体をdelete

### External Reference Artifact

親が既に保存した成果物をURI metadataで登録。

- CoreはURIをfetchしない
- Core retentionで外部実体をdeleteしない
- External LLM `task_submit.artifacts` はこの形式のみ

ArtifactStore backendはabstract、MVP標準`LocalArtifactStore`。

## 14. Dynamic Job

Root scalar `foreach`、Nested `foreach.parent/items`。

- parentはDAG dependency edge
- fixed depth limit無し
- generated Job default max1000 / Workflow Run
- expansion all-or-nothing
- stable key推奨
- full logical `job_key`はparent path込み
- fixed byte length limit無し
- filesystem pathはopaque ID
- group status running/completed

## 15. Reusable Workflow

`uses`でChild Workflow Runを呼ぶ。

Relative fileはcaller Workflow source directory基準、Workflow root外escape reject。Non-filesystem callerはregistered ID。

Parent Job最初のactivationでChild bindingを固定しRetryでも同binding。

Parent/Child state非共有。Input/Output/Artifactのみ明示data flow。

Child Runへのpublic direct pause/resume/cancel/retry/priority updateは禁止。

## 16. External LLM / Human

External:

- ready時Attempt + Task
- Lease default60分、expiry default`requeue`
- atomic claim/lease owner submit
- resultは任意JSONでPayloadStore利用
- `claim_next`は通常claimと同ordering

Human:

- ready時Attempt + pending Review
- approve/reject
- lease/timeout無し
- concurrent submit first-wins
- standard Job Outputはnull、Review metadataはReview API

## 17. Retry / Recovery

Retryはnew Attempt。Retry targetのpersistent Input/Definition/Action version/Dynamic iteration/Reusable bindingは固定。

Automatic retryは明示時のみ。

Manual retryはfailed Job指定。ただし **過去Attemptとpersistent Input Snapshotが存在するJobだけ**を対象にする。Activation/Input resolution段階でAttempt開始前にfailedとなりInput Snapshotが存在しないJobは `retry_input_unavailable` としてsame Run retryを拒否し、新Workflow Runを要求する。

Completed/failure Runをmanual retryする場合は同じRunをreopenし`run_attempt++`。Success/cancelled Runはretry不可。

Recoveryだけでcompleted Runをreopenしない。

## 18. Same-Run Result Reuse

自動result reuseは**同一Workflow Run内だけ**。Cross-Run/global cache無し。

Reuse判定キー:

- final persistent Job Input
- direct upstream Artifact identities
- entire Workflow Definition hash
- executor/action identity + version

ActionがRuntime `state.get`やfrozen dependency外Artifactをdynamic materializeした場合はautomatic reuse不適格。

Manual Retry後:

- blocked/skipped descendantsはdependency/conditionを再評価
- successful descendantsはreuse keyを再検証
- matchなら既存success維持
- mismatch/ineligible/Payload欠落なら`successful_job_result_not_reusable`でfail-closedし新Workflow Runを要求

MVPではsuccess Jobをchanged Inputで自動再実行しない。

## 19. Pause / Cancel

Pause:

- running internal継続
- new internal/External claim/Dynamic expansion停止
- existing External submit/Human submit/started Child進行可

Cancel:

- new activation禁止
- queued/waiting cancel
- active External Lease invalidated
- running internal graceful cancel
- Childへpropagate

## 20. Workflow State / Log

Mutable state: persistent get/set、last-write-wins、revision/history。CAS/increment無し。Child independent。

Execution Log本文はfile。Event Logはappend-only DB。Stepは観測単位のみ。

Attempt temp directoryは終了時削除、sandboxではない。

## 21. Persistence / Idempotency

Standard SQLite、WAL/FK/busy timeout。

重要constraint/transaction:

- one running internal / Workflow Run
- Dynamic expansion root/nested unique
- one Child / Parent Attempt
- one External Task / Attempt
- one active Lease / Task
- one Human Review / Attempt
- state current+history atomic
- concurrency holder atomic
- Payload inline/blob metadata integrity
- reuse metadata/pending state

Idempotency scopeはsystem namespace + resource + AccessScope + Actor/client principal。Default TTL24h。TTL後expired rowが残っていてもtransaction内で置換しsame keyをnew requestとして利用可能。

## 22. Authorization / Service / MCP

認証は親責任。Coreは全public read/writeでAuthorizationProviderを通す。Default AllowAll。

Web/MCP/Pythonは同じService layer。

MCP public name:

```text
<namespace>_wf_*
```

主要:

```text
wf_start/list/info/pause/resume/cancel/retry/priority_update
wf_output_info/read
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_runner_info
```

`wf_info`はOutput本文/Execution Log本文を含めない。Output本文は`wf_output_read`で取得し、optional JMESPath `select`で巨大JSONの一部だけを読める。MCP response上限超過時はsilent truncateせずerror。

## 23. Retention / Scheduler / CLI / WebUI

Retention default無期限。

Scheduler/Cron/CLIはMVP無し。

WebUI画面設計は後続。ただしService/HTTP Adapter境界は最初から持つ。

## 24. Source of Truth

実装時は`docs/detailed-design/01`〜`13`をSource of Truthとする。基本設計と詳細設計が衝突する場合は最新の具体的詳細設計を優先し、基本設計も同期修正する。
