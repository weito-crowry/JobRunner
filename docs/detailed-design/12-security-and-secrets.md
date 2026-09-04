# 12. Security / Secrets 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/04-runner-and-ipc.md`
  - `docs/detailed-design/08-persistence.md`
  - `docs/detailed-design/11-service-api-and-mcp.md`

## 1. 目的

本書は JobRunner Core の認可境界、ActorContext、AccessScope、AuthorizationProvider、Secrets注入、ログ保護、Runner/Action Process間の機密情報の扱いを定義する。

## 2. 基本原則

1. 認証そのものは親システムの責任。
2. CoreはAuthorization hookを最初から持つ。
3. MVP既定providerはAllowAll。
4. AllowAllでもService operationはAuthorizationProviderを必ず通す。
5. SecretsはWorkflow YAMLへ値を保存しない。
6. SecretsはSQLite / Event Log / Execution Logへ平文保存しない。
7. Action Runnerへ渡すSecretsは必要最小限にする。
8. CoreはMVPで本格sandboxを提供しない。
9. 任意コード実行の安全性は専用Action / 親システム責任。

## 3. Trust boundary

概念:

```text
External client
  ↓
Parent authentication
  ↓ ActorContext
JobRunner Adapter
  ↓
AuthorizationProvider
  ↓
Runtime Service
  ↓
Persistence / Runner / Action
```

JobRunner Coreは親システムが供給したidentityを信頼する。

## 4. ActorContext

概念型:

```text
actor_type
actor_id optional
source optional
claims optional
metadata optional
```

例:

```text
user
service
agent
system
runner
```

## 5. AccessScope

Coreが操作対象を絞り込める抽象scopeを持つ。

概念例:

```text
project_id optional
workspace_id optional
tenant_id optional
resource_tags optional
```

Core自体がproject/tenantの意味を固定しない。

親システムがAccessScopeの意味を定義する。

## 6. AuthorizationProvider

概念interface:

```python
class AuthorizationProvider:
    def authorize(self, actor, operation, resource, scope) -> AuthorizationDecision:
        ...
```

decision:

```text
allowed
reason optional
filtered_scope optional
```

## 7. AllowAll

MVP default:

```text
AllowAllAuthorizationProvider
```

ただしService codeでprovider呼び出しを省略しない。

将来provider差し替えだけで制限を追加できる構造にする。

## 8. Authorization対象

少なくとも以下を通す。

```text
workflow start/read/list
pause/resume/cancel/retry
priority update
external claim/submit
human review submit
artifact metadata read
log read
runner info read
state read/write via service
```

Action Runtime Handle経由のstate/artifact操作もcurrent Workflow Run scopeを引き継ぐ。

## 9. Worker/Runner identity

RunnerからCoreへの更新は外部ユーザー認証ではなく、runtime内部のidentityでfencingする。

最低限:

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id
```

current ownershipと一致しないlate updateを拒否する。

## 10. Secrets Provider

親システムがSecrets ProviderをRuntime初期化時に渡す。

概念:

```python
class SecretsProvider:
    def get(self, name, actor, scope):
        ...
```

MVPではdict/env wrapperでもよい。

## 11. YAML secrets参照

値を直接書かず参照する。

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

Workflow Run snapshotには参照式を保存し、解決後のsecret valueを保存しない。

## 12. Secret解決タイミング

Job Inputの永続snapshotを作る際、Secret値を通常Input JSONへ埋め込まない。

永続Inputとruntime-only secret injectionを分離する。

Action起動直前に必要secretを解決し、Action Runnerへ渡す。

## 13. Secret allow/request set

Actionが必要なSecret名をmetadataで宣言できる構造を推奨する。

例:

```python
registry.register(
    "llm.call",
    version="1",
    secrets=["OPENAI_API_KEY"],
    callable=call_llm,
)
```

YAMLから任意secret名を自由列挙させる場合もAuthorization / parent policyで制限可能にする。

MVPでは厳密allow-listを必須としないが拡張点を持つ。

## 14. IPCとSecrets

Runner -> Action Runner IPCでSecretを渡す必要がある場合、専用start payloadのruntime-only fieldとして渡す。

以下へ書かない。

- IPC debug dump
- Event payload
- normal Execution Log
- SQLite Input snapshot

## 15. Secret redaction

Execution Logへの漏洩低減として、既知Secret値をredactできるhookを持つ。

ただし完全な漏洩防止を保証しない。

Actionが意図的に変形・分割して出力したSecretをCoreが完全検出する責任は負わない。

## 16. Environment variables

親システムがSecretsをOS environment経由で供給するAdapterを作ってもよい。

Coreは親Process全environmentを無条件にActionへ引き継がない。

Action Runnerのenvironmentは必要最小限に構築することを推奨する。

## 17. Execution Log

Log readはAuthorization対象。

Log pathを外部APIへ直接filesystem pathとして公開する必要はない。

Serviceはattempt_id経由でreadする。

Path traversalを防ぐため、外部入力から任意pathをreadしない。

## 18. Artifact URI

Artifact `uri` は参照情報。

Coreは任意URIを自動fetchしない。

したがってSSRF等の外部fetch責務をCore Artifact subsystemに持ち込まない。

Action / 親システムがURIを解釈する。

## 19. Workflow YAML

YAMLに任意Python/Shellコードを埋め込まない。

Action IDから登録済みcallableのみ実行する。

未登録Actionは開始前validationで拒否。

## 20. Arbitrary code execution

Core標準機能として以下を提供しない。

```text
shell: <arbitrary command>
python: <arbitrary source>
```

必要な親は専用Actionを登録する。

Sandboxが必要なら専用Action内でDocker等を利用する。

## 21. Temp directory

Attempt専用directoryは整理目的でありsecurity sandboxではない。

ActionはOS権限上アクセス可能な他pathへ到達できる可能性がある。

この制約をドキュメントで明記する。

## 22. SQLite

SQLite fileのOS file permissionは親システムdeployment責任。

CoreはDB暗号化をMVP標準要件にしない。

Secret値をDBへ保存しないことでriskを下げる。

## 23. MCP exposure

親システムは公開するMCP toolを選択可能。

例えばread-only deploymentではstart/cancel/retryを登録しない運用を可能にする。

Tool namespaceを必須化し、複数親システム間の誤操作を減らす。

## 24. Human / External actor

External task `claimed_by` / Human review actorをEventへ保存する。

個人情報等をどこまで保存するかは親側ActorContext設計に従う。

Coreは不要なidentity属性を自動収集しない。

## 25. Child Workflow

Child Workflow Runは親ActorContext / AccessScopeをsnapshotまたは参照可能な形で継承する。

権限拡大しない。

Child側でより狭いscopeへ制限することは許可する。

## 26. Error response

Authorization failureで内部存在情報を過剰に漏らさない。

Adapterは`forbidden` / `not_found`の使い分けをProvider policyに従える。

## 27. Audit Event

state changeにはactor/sourceを記録する。

代表:

```text
workflow_started
workflow_cancel_requested
retry_requested
external_task_claimed
external_task_submitted
human_review_submitted
priority_updated
```

Secret値をpayloadへ含めない。

## 28. Security拡張点

将来候補:

```text
RBAC provider
ABAC provider
PostgreSQL RLS adapter
Secret manager adapter
Docker sandbox Action
OS-level runner isolation
signed external task claims
```

MVP Coreをこれらに依存させない。

## 29. 受入条件

1. 全state-changing ServiceがAuthorizationProvider通過
2. AllowAll default
3. deny providerで操作拒否
4. AccessScope filtering
5. Secret YAML参照
6. Workflow snapshotにSecret値なし
7. DB InputにSecret値なし
8. EventにSecret値なし
9. Execution Log redaction hook
10. late Runner fencing
11. MCP tool選択公開
12. arbitrary shell非搭載
13. Artifact URI自動fetchなし
14. log path traversal拒否
15. Child権限非拡大
