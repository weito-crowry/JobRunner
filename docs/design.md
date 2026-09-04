# JobRunner 基本設計

- Status: Draft v0.4
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
- Artifact metadata
- Human Review
- External LLM pull execution
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log
- SQLite persistence
- MCP / Web / Python 共通Service

非目標: Airflow/Temporal/n8n級の独立大規模基盤、Kubernetes/distributed scheduler、任意shell標準機能、GUI workflow editor、CLI/Cron、通知、完全GitHub Actions互換、本格sandbox、認証基盤。

## 2. 用語

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
| Artifact | 成果物の参照metadata |

正式用語は `Runner`。`Worker` は使わない。

## 3. 組み込み構成

```text
Parent System
├─ JobRunner Runtime / Service
├─ Action Registry
├─ SQLite
├─ MCP Adapter optional
├─ Web Adapter optional
└─ Runner Pools
   └─ Runner Process
      ├─ Heartbeat Thread
      └─ Common Action Runner Process
```

親起動時にRuntime/Runner Poolを自動初期化。JobRunner専用サービスの手動起動は不要。

1 Repository / 1 Python Package。MCP/Webはoptional dependency。

## 4. Workflow Definition

Canonical formatはYAML。safe loader、duplicate key/merge key/custom tag拒否、unknown key拒否。

Run開始時に実使用定義をsnapshot:

- Workflow ID/version
- source YAML
- canonical definition JSON/hash
- Input snapshot
- Action ID/version
- optional `source_identity`

既存Runは元YAML変更の影響を受けない。

## 5. YAML / Expression

GitHub Actions風の `${{ ... }}`。

- CEL: condition/value計算
- JMESPath: JSON filter/projection

主要context: `inputs/needs/env/state/item/iteration/failure/outputs/workflow/run/job/jobs`。

Secretsは **internal Action Jobの`with`だけ**。`env`, external/human/reusable, condition等では禁止。

## 6. Action Registry

親側bootstrapでProcess起動時に再構築。DBへcallableを保存しない。

Actionは `action_id + version`。sync/async対応。

Runtime Handle optional:

- progress
- step/log
- state get/set
- cancel check
- Artifact登録

Retry/ResumeでAction version mismatchならfail-closed。

## 7. Runner / Runner Pool

Poolは親が事前登録。未登録はRun開始前error。

`runs-on`省略時はSystem `default_runner_pool`、既定文字列 `default`。

Runnerは常駐しJobをpull。Runnerが取得する単位はJob。

**同一Workflow Runのinternal Jobは同時に最大1件。** 別Workflow Runは並列可能。

選択順:

1. Workflow Run priority
2. Job priority
3. Dynamic order rank
4. source order
5. ready time
6. deterministic ID tie-break

External task claimも同じ順序軸を使う。

## 8. Runner / Action Process / IPC

Runner管理ProcessとAction子Processを分離。HeartbeatはRunner内別Thread。

既定:

- Heartbeat: 5秒
- Runner lost: 20秒

Common Action RunnerがRegistry bootstrap後 `action_id + version`を解決。

Runner ↔ Action RunnerはJSON Lines v1。stdout/stderrはprotocolと分離してExecution Logへ。

Action ProcessはCore DBを直接触らない。

Parent正常shutdownはWorkflow cancelではない。未完了running Attemptは次回Runtime起動時に旧Runtimeのorphanとして通常 `runner_lost` Recoveryへ流す。

## 9. Timeout

Hidden default timeoutなし。

`timeout-minutes` は **internal Jobのみ**。

External LLMはLease、Human ReviewはMVPで期限なし、Reusable WorkflowはChild内Jobのtimeoutで制御。

## 10. 状態

Job status:

```text
queued
running
waiting_external
waiting_review
waiting_child
completed
```

Conclusion:

```text
success
failure
cancelled
skipped
blocked
```

Workflow Run status:

```text
queued
running
paused
completed
```

Conclusion:

```text
success
failure
cancelled
```

## 11. 条件

`if`未指定は `success()`。

Nested Dynamicでは `foreach.parent` も暗黙required dependencyとしてhelper評価に含む。

Effective success:

- `success`
- `failure` + `continue-on-error=true`

**通常Jobの`skipped`はeffective successではない。** そのJobに依存する後段Jobは明示条件が無ければ既定でskipする。

Dynamic template groupは個別generated Jobのskipをgroup集約した結果として `success` になり得る。

`failure()/cancelled()/always()`も同じdependency setで評価する。

Workflow cancel後、`always()`でも新Job/expansionを開始しない。

## 12. Dynamic Job

Root:

```yaml
foreach: ${{ needs.generate.outputs.items }}
```

Nested:

```yaml
foreach:
  parent: evaluate
  items: ${{ iteration.parent.outputs.items }}
```

`parent`はDAG dependency edge。

- 入れ子深さ固定上限なし
- generated Job既定上限1000 / Workflow Run
- expansion all-or-nothing
- stable raw key推奨
- full logical `job_key`は親pathを含む
- `job_key`に固定byte長上限はMVPでは置かない
- filesystem pathにはopaque internal IDを使う

Dynamic group statusは未完了=`running`、全expansion/job terminal=`completed`。0件は `completed/success`。

## 13. Reusable Workflow

`uses`で呼ぶ。Childは独立Workflow Run。

Relative file referenceは **caller Workflow source fileのdirectory基準**。Workflow root外へ出るpath/symlinkは拒否。source directoryを持たないcallerはregistered Workflow IDを使用。

Parent Job最初のactivationでChild Definition bindingを固定。Retryは同じbindingからnew Child Runを作る。

Parent/Child stateは共有しない。Input/Output/Artifactのみ明示mapping。

MVPでChild Runへのpublic direct pause/resume/cancel/retry/priority updateは禁止。

## 14. External LLM

Runner不使用。ready時にAttempt + External Task。

Lease default 60分、expiry default `requeue`。

`task_claim` atomic exactly-one。Lease ownerだけsubmit可能。stale/expired/cancelled submit拒否。

`requeue` expiryはsame Attemptを再claim。`fail`はAttempt failure -> Retry policy。

`claim_next`はsubmit後別transactionで通常claimと同じselection ruleを使う。

## 15. Human Review

Runner不使用。ready時にAttempt + pending Review。

Outcomeは `approve | reject`。

Lease/timeoutなし。Concurrent submitはfirst-wins。

Rejectはfailure。人間操作でfailed Jobを直接successへ変更しない。

## 16. Retry / Recovery

Retryはnew Attempt。Input/Definition/Action version/Dynamic iteration/Reusable bindingを固定。

Automatic retryは明示時のみ。Manual retryはfailed Job指定。

Completed/failure Runをmanual retryすると同じRunを明示reopenし `run_attempt` を増やす。success/cancelled Runはretry不可。

Recoveryだけでcompleted Runをreopenしない。

## 17. Pause / Cancel

Pause:

- running internal継続
- new internal claim/Dynamic expansion/External claimを止める
- existing External submit/Human submitは受理
- started Childは継続

Cancel:

- new activation禁止
- queued/waitingをcancel
- active External Lease invalidated
- running internalへgraceful cancel
- Childへ伝播

## 18. Output / Artifact / State / Log

Job OutputはJSON-compatible、既定最大4MiB。大きい実体はArtifactへ。

Artifact実体は親/Action責任。Coreはimmutable metadata/historyを管理。

Workflow mutable stateはpersistent get/set + revision history。last-write-wins。CAS/incrementはMVPなし。

Execution Log本文はfile、Event Logはappend-only DB。

Attempt temp directoryは終了時削除。Security sandboxではない。

## 19. SecretGuard

Current internal Attemptでmaterializeした既知Secret値は永続化前に検査する。

- Output / State / Artifact metadata / Event / error details: exact substring検出時は `secret_value_persistence_blocked` でfail-closed
- stdout/stderr/log: `[REDACTED]` へ置換
- Base64/hash/分割等の変形Secretは完全検出を保証しない

## 20. Persistence

標準SQLite専用DB、WAL/FK/busy timeout。

重要unique/transaction:

- one running internal / Workflow Run
- Dynamic expansion root/nested uniqueness
- one Child / Parent Attempt
- one External Task / Attempt
- one active Lease / Task
- one Human Review / Attempt
- state current + history atomic
- Workflow concurrency start/release atomic

## 21. Idempotency

state-changing operationはoptional `request_id`。

Scopeは:

- system namespace
- resource scope
- AccessScope identity
- Actor/client principal

を含む。

Default TTL 24h。

TTL内same hashはreplay、different hashはconflict。TTL後はexpired DB rowが残っていてもtransaction内で置換してsame keyを新requestとして再利用可能。

## 22. Authorization / Security

認証は親責任。Coreは全public read/writeでAuthorizationProviderを通す。Default AllowAll。

SecretsはDB/Event/Execution Logへ平文保存しない。Action Runnerへ必要最小限をruntime-only注入。

Core標準で任意shell/source実行や本格sandboxを提供しない。

## 23. Service / MCP

Web/MCP/Pythonは同じService layer。

MCP public name:

```text
<namespace>_wf_*
```

主要操作:

```text
wf_start/list/info/pause/resume/cancel/retry/priority_update
wf_task_info/claim/submit
wf_review_list/info/submit
wf_artifact_info
wf_log_read
wf_runner_info
```

Child direct controlは禁止。

## 24. Retention / Scheduler / CLI / WebUI

Retention既定無期限。

Scheduler/Cron/CLIはMVPなし。

WebUI画面設計は後続。ただしService/HTTP Adapter境界は最初から持つ。

## 25. 詳細設計

`docs/detailed-design/01`〜`13` を実装時Source of Truthとする。基本設計と詳細設計が衝突する場合は、より具体的な最新詳細設計を優先し、基本設計も同時に同期修正する。
