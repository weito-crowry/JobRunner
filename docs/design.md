# JobRunner 基本設計

- Status: Draft v0.2
- 対象: MVP
- WebUI: 画面詳細設計のみ別途
- Source of Truth: 本書 + `docs/detailed-design/01`〜`13`
- 用語: GitHub Actionsに対応概念がある場合は可能な限り同じ名称

## 1. 目的

JobRunnerは既存アプリケーションへ組み込む軽量なPersistent Workflow / Job Runtime。

提供:

- Workflow / Workflow Run
- Job / Attempt / Step
- `needs`
- Action Registry
- Runner / Runner Pool
- Retry / Resume / Pause / Cancel
- Dynamic Job / Reusable Workflow
- Artifact reference / Workflow state
- External LLM pull executor / Human Review
- Event / Execution Log / Progress
- SQLite persistence / Recovery
- Python Service / MCP / Web Adapter

Airflow/Temporal/n8n規模の独立分散基盤は目標にしない。

## 2. 非目標

MVPでは:

- GUI Workflow editor
- SaaS connector marketplace
- Core標準 arbitrary shell/python execution
- remote arbitrary Runner / multi-machine scheduler
- Kubernetes / CPU-RAM-GPU resource scheduler
- Cron/Scheduler
- CLI
- Notification connector
- custom expression language
- GitHub Actions完全互換
- OS-level sandbox
- Authentication/user management
- source/dependency/environment完全archive

を行わない。

## 3. 正式用語

| 用語 | 意味 |
| --- | --- |
| Workflow | YAMLの設計図 |
| Workflow Run | Workflowを1回起動した実行実体 |
| Job | Workflow内の実行単位 |
| Attempt | Jobの各execution試行 |
| Step | Job内の観測/Log/Progress単位 |
| Action | 親がRegistryへ登録する実処理 |
| Runner | internal Jobを実行する常駐Process |
| Runner Pool | Runnerのgroup |
| Artifact | Job成果物への参照metadata |

正式用語として`Worker`は使わない。

## 4. 組み込み構成

```text
Parent System
├─ JobRunner Runtime / Services
├─ Action Registry
├─ SQLite
├─ MCP Adapter optional
├─ Web Adapter optional
└─ Runner Pool Supervisor
   ├─ Runner Process
   │  └─ Common Action Runner Process
   └─ ...
```

- 親起動でRuntime/Runner Pool自動初期化。
- JobRunnerを別サービスとして利用者が手動起動する前提にしない。
- CoreはMCP/Webへ依存しない。
- 1 repo / 1 Python package。

配布想定:

```text
jobrunner
jobrunner[mcp]
jobrunner[web]
jobrunner[all]
```

## 5. Workflow Definition

- authoring/storage canonicalはYAML。
- safe YAML loader、duplicate mapping key/merge key/custom tag禁止。
- strict schema、unknown field拒否。
- typed immutable modelへ正規化。
- 任意codeをYAMLへ埋め込まない。
- ActionはRegistryのIDで参照。

Workflow Run開始時に:

```text
source YAML
canonical typed definition
SHA-256 definition_hash
Workflow Input
Action ID/version
source_identity optional
```

をsnapshotする。既存Runは元YAML変更の影響を受けない。

## 6. YAML / Expression

GitHub Actions風:

```yaml
name: Strategy Evaluation
version: 1

inputs:
  symbol: {type: string, required: true}

jobs:
  generate:
    action: fx.generate
    with:
      symbol: ${{ inputs.symbol }}

  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.items }}
    key: ${{ item.id }}
    action: fx.evaluate
```

- CEL: condition/value計算
- JMESPath: JSON filter/projection
- wrapper `${{ ... }}`
- MVP Python >=3.10

Job Output canonical JSON既定上限は4 MiB。超える実体は親/Action側で保存しArtifact参照にする。

## 7. Job / Workflow status

Workflow Run status:

```text
queued | running | paused | completed
```

conclusion:

```text
success | failure | cancelled
```

Job status:

```text
queued | running | waiting_external | waiting_review | waiting_child | completed
```

conclusion:

```text
success | failure | cancelled | skipped | blocked
```

PauseはWorkflow Run側のScheduling gate。Jobに`paused` statusは持たない。

## 8. Action Registry

親の共通bootstrapで各ProcessがRegistryを再構築する。

```python
registry.register("fx.run_backtest", version="3", callable=run_backtest)
```

- sync/async対応
- simple: `action(input_data) -> dict`
- optional Runtime Handle: Log/Progress/Step/State/Artifact/Cancel
- Action version更新は親責任
- Coreはsource code/dependencyを解析・保存しない

Retry/ResumeはRun snapshot/bindingのAction versionを要求し、current Registryが提供できなければfail-closed。

## 9. Runner / Runner Pool

- 親起動時にRunner Process常駐。
- RunnerはJobをpull。
- Workflow RunへRunner固定なし。
- external/human/child waitでRunner解放。
- 同一Workflow Run internal Job同時実行最大1。
- 別Workflow Runは複数Runnerで並列。
- Poolは親登録済みだけ使用。
- `runs-on`省略internal JobはSystem `default_runner_pool`（既定名`default`）へ解決。
- PoolごとのAction allow-listなし。
- CPU/RAM/GPU高度schedulerなし。

選択順:

1. Workflow Run priority
2. Job priority
3. Dynamic `order_by`
4. source order
5. ready time
6. deterministic ID tie-break

## 10. Runner Process / Heartbeat

FX-LLMで実績のある構造を踏襲:

```text
Runner Process
├─ internal RunnerService -> same SQLite
├─ main loop
├─ Heartbeat Thread
└─ Common Action Runner child
```

Actionは子Process。Runner main loopとHeartbeatをAction負荷から分離。

既定:

```text
heartbeat 5s
runner lost 20s
main-loop stale 15s
```

Heartbeatはmain-loop tickがfreshなときだけ更新し、Heartbeat threadだけ生き残る偽陽性を防ぐ。

必須のRunner HTTP/Broker serviceは置かず、Runner Processは同じSQLiteへ内部Service/Repository経由で短いtransactionを行う。Action Runner childはDBへ直接アクセスしない。

## 11. Runner ↔ Action Runner IPC

local JSON Lines `jobrunner.action-ipc.v1`。

Runner -> child:

```text
start / cancel_requested
```

child -> Runner:

```text
ready/log/progress/step/artifact/state/result/error/exiting
```

Structured IPCとAction stdout/stderrはchannel分離。stdout/stderrはExecution Logへ。

Attempt temp directoryは内部IDで作り終了時削除。Sandboxではない。

## 12. Retry

- default manual Retry。
- Automatic RetryはJob YAMLで明示。
- Retryはnew Attempt。
- Automatic backoff予約時はAttempt rowを作らず、実際のexecution activationで作る。
- persisted Job Input / Definition / Dynamic context / Reusable binding固定。
- internal Secret referenceは固定だがvalueは各Attempt直前に再materialize。
- Job timeout未指定は無期限。

Manual RetryはMVP Job指定のみ。completed failed Workflow Runのfailed Jobを明示Retryすると同じRunをreopenし`run_attempt`を増やす。Recoveryだけではterminal Runをreopenしない。

## 13. Cancel / Pause

Cancelはgraceful。

- queued/waitingをcancel
- active external Lease invalidation
- running internalへcancel request
- Childへcancel propagation
- new activation禁止
- late submit/update拒否

Public force-success/force-kill無し。

Pause:

- running internal継続
- new internal/external/new Child開始なし
- existing external submit/Human submit/started Child進行は受理

## 14. Failure / Conditions

Structured failure:

```text
category/code/message/retryable/details/occurred_at
```

`continue-on-error`はJob activation時boolean snapshot。Job failureを書き換えず、dependency/Workflow conclusionでeffective successとして扱う。

Helper:

- `success()`: needsが全てsuccessまたはallowed failure
- `failure()`: non-allowed failure/blockedあり
- `cancelled()`: Run cancelまたはneed cancelled
- `always()`: declared needs terminal

Workflow cancel後は`always()`でも新Job開始しない。

## 15. Inputs / State / Secrets

Workflow Inputはtyped/schema、Run start snapshot。

Job `with`はfield mapping + `$base` shallow copy/override。

Workflow共有値:

- `env`: immutable
- mutable state: persistent get/set + last-write-wins + revision/history

Child Workflow stateは独立。

Secrets:

- 親Providerからruntime注入
- `${{ secrets.* }}`はinternal Action Job `with`のみ
- External/Human/Reusable parent/conditionsには禁止
- DB/Event/Logへ平文保存しない

## 16. Artifact / Log / Step / Progress

Artifact実体は親/Action側保存。Coreはmetadata/history/current successful Attempt referenceのみ。

```yaml
source: ${{ needs.export.artifacts.dataset }}
```

Execution Logはfile、DBにはmetadata。Event Logはappend-only structured audit。

Stepは観測単位で独立Retry/needs/Artifact ownershipなし。

ProgressはAction報告 + Workflow自動算出可能。

## 17. Dynamic Job

MVP:

- Root `foreach`
- Nested `foreach.parent/items`
- stable `key`
- full logical path identity
- arbitrary nesting depth
- generated Job既定上限1000/Workflow Run
- all-or-nothing expansion
- `order_by`
- group `needs`

Root:

```text
evaluate[candidate_a]
```

Nested:

```text
evaluate[candidate_a]/condition[x]
```

Group Output/Artifactはfull logical job_key map。

## 18. Reusable Workflow

```yaml
jobs:
  child:
    uses: ./workflows/child.yml
    with: {}
```

- Childは独立Workflow Run
- input/output/artifact明示mapping
- cycle detection
- state非共有
- Parent cancel伝播
- Parent Pauseは開始済みChildへ非伝播
- public Child controlはMVP拒否/read可
- first Parent Job activationでReusable Bindingをsnapshotし、Retryは同bindingからnew Child Run

Workflow top-level `outputs`は`${{ jobs.<job>... }}`で評価。

## 19. External LLM / Human

External:

- `executor: external_llm`
- Runnerなし
- atomic claim + Lease
- default 60min
- expiry `requeue|fail`、default requeue
- requeueは同Attemptでnew Lease
- submitはvalid current Lease必須
- optional Artifact refs / `claim_next`

Human:

- `executor: human`
- pending Review row
- outcome approve/reject
- Leaseなし
- concurrent submit first wins
- failed->success手動overrideなし、Retry

## 20. Persistence

SQLite専用DB既定。WAL + foreign keys + busy timeout。

重要constraint:

- 1 Workflow Run running internal Job最大1 partial unique index
- Dynamic expansion root/nested unique
- Reusable binding one/Parent Job
- Child Run one/Parent Attempt
- External Task one/Attempt + active Lease one/Task
- Human Review one/Attempt
- State revision unique
- idempotency unique

IDはtype prefix + UUID4 hex。

Persistence interfaceは将来PostgreSQLへ交換可能。

## 21. Authorization / Idempotency

Authenticationは親責任。

Core:

```text
ActorContext
AccessScope
AuthorizationProvider
```

MVP default AllowAll。ただし**全public read/write Service operationがhookを通る**。

State-changing requestはoptional idempotency `request_id`。side effect/Event/resultは可能な限り同一transaction。TTL既定24h。

## 22. Service / MCP

Definition/Runを分ける:

```text
wf_definition_list/info
wf_start
wf_run_list/info
wf_pause/resume/cancel/retry/priority_update
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_runner_info
```

MCP public name:

```text
<system_namespace>_<logical_name>
```

例 `novel_wf_start`, `fx_wf_task_claim`。

親は不要toolを公開しない。

## 23. Retention / Scheduler / Notification / CLI

Retention機能あり、既定無期限。Runtime startup/maintenance hook/明示Serviceから実行可能。

MVPでScheduler/Cron/Notification/CLIなし。外部Schedulerは安定したstart APIを呼べる。

## 24. Static validation

YAML load + Workflow Run startの2段階でstrict validation。

少なくとも:

```text
YAML duplicate/tag/merge
schema/unknown field
Input
needs/cycle
Action/version
Runner Pool
CEL/JMESPath
Dynamic parent/key/order/limit
Reusable reference/cycle
retry/timeout/external/concurrency
Secret使用位置
```

推測で補正しない。

## 25. WebUI

画面詳細は別設計。方向性はHuman control panelとして十分作り込む。

Runtime ServiceはWebUIに必要なRun/Job/Attempt/Step/Log/Artifact/Human Review/Retry/Resume/Cancel/Startを提供する。

## 26. 詳細設計一覧

```text
01-workflow-definition.md
02-expression-and-inputs.md
03-runtime-and-scheduling.md
04-runner-and-ipc.md
05-dynamic-jobs.md
06-reusable-workflows.md
07-external-and-human.md
08-persistence.md
09-artifacts-logs-state.md
10-retry-recovery-cancel.md
11-service-api-and-mcp.md
12-security-and-secrets.md
13-testing.md
```
