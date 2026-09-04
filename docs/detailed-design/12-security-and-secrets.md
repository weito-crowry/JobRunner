# 12. Security / Secrets 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `04-runner-and-ipc.md`, `08-persistence.md`, `11-service-api-and-mcp.md`

## 1. 基本原則

1. Authenticationは親システム責任。
2. CoreはAuthorizationProviderを必須hookとして持つ。
3. DefaultはAllowAll。
4. 全public Service read/write operationがAuthorizationProviderを通る。
5. Secret valueをDefinition/SQLite/Event/Execution Logへ平文保存しない。
6. Secretはinternal Action Job `with`だけへruntime materialize。
7. Coreは本格sandbox/任意code executionを標準提供しない。
8. Runner内部identityはfencingで守る。

## 2. Trust boundary

```text
External caller
 -> Parent authentication
 -> ActorContext/AccessScope
 -> Adapter
 -> AuthorizationProvider
 -> Public Service
 -> Persistence

Parent Supervisor
 -> Runner identity/fencing
 -> Internal RunnerService
 -> Persistence

Runner -> Action Runner -> registered Action
```

Action Runner childはDBへ直接接続しない。

## 3. Actor / AccessScope / Authorization

ActorContext:

```text
actor_type
actor_id optional
source optional
claims optional
metadata optional
```

AccessScopeは project/workspace/tenant/resource tags等の抽象scope。意味は親が定義。

AuthorizationProvider:

```python
class AuthorizationProvider:
    def authorize(self, actor, operation, resource, scope) -> AuthorizationDecision:
        ...
```

Read含む全public operationを通す。Listではfiltered scopeをqueryへ適用。

ChildはParent scopeを継承し権限拡大しない。

## 4. Runner fencing

Runner updateは:

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id where applicable
```

をcurrent ownershipと照合。旧instanceのlate update拒否。

## 5. SecretsProvider

```python
class SecretsProvider:
    def get(self, name, actor, scope):
        ...
```

親側dict/env/secret manager adapterでよい。

Secret name syntax:

```text
^[A-Za-z_][A-Za-z0-9_]*$
```

## 6. Secret reference policy

許可:

```yaml
jobs:
  call_api:
    action: api.call
    with:
      token: ${{ secrets.API_TOKEN }}
```

**internal Action Job `with`のみ。**

禁止:

```text
env
External LLM with
Human with
Reusable Workflow parent with
if
success_if
retry if
continue-on-error
foreach/key/order_by
Workflow outputs
concurrency
```

Childは自分のinternal ActionからSecretを参照する。

## 7. Materialization / Retry

Persistent Job InputにはSecret reference markerだけ。各Attempt Action起動直前にmaterialize。

Retryでreference固定、rotation後のnew value使用は許容。

missing Secretはchild起動前 `secret_not_found` failure。

## 8. Persistence-bound Secret Guard

「Secretを保存しない」を実装可能にするため、current AttemptでmaterializeしたSecret値を `SecretGuard` に登録する。

### 8.1 Structured persistence

以下をSQLite等へ永続化する直前に、JSON-compatible value内のstring scalarを再帰走査する。

- Action Output
- Runtime Handle `state.set` value
- Artifact `uri` / `metadata`
- structured Event payload
- structured error/details

既知Secret値が **string全体または部分文字列として含まれる** 場合、永続化を拒否する。

failure:

```text
category=security
code=secret_value_persistence_blocked
retryable=false
```

Secret valueが短すぎて誤検知しやすい場合も、参照されたSecretである以上fail-closedを優先する。

### 8.2 Execution Log

Log/stdout/stderrは永続化を拒否するとActionの診断不能になり得るため、既知Secret値を `[REDACTED]` に置換してからwriteする。

### 8.3 IPC debug / idempotency

Raw IPC debug dumpは標準で保存しない。Idempotency hash/resultへSecret値を入れない。

### 8.4 限界

Base64/hash/分割/暗号化等で変形されたSecretを完全検出する責任はCoreに負わせない。Coreが知るmaterialized値のexact substring検出までを保証する。

## 9. Secret allow set

Action registration metadataで必要Secret名を宣言可能。MVPで必須ではないが親policyが参照可能Secret名を制限できるhookを持つ。

## 10. Process environment

親Process全environmentをAction Runnerへ無条件継承しない。

Child environmentはProcess起動に必要な最小非Secret値を基本とし、Secretはruntime-only IPC/専用注入。

## 11. YAML / Reusable security

Safe loader:

- custom/object tag reject
- duplicate key reject
- merge key reject
- arbitrary include/fetch reject

Reusable `uses`はlocal Workflow root/registered IDのみ。URL fetch無し。

## 12. Arbitrary code / Sandbox

Core標準で `shell:` / arbitrary Python sourceを提供しない。

必要なら親が専用Actionを登録し、Docker等で隔離する。

Attempt temp directoryはsandboxではない。

## 13. Log / Artifact / filesystem

Log readはAuthorization対象。外部path指定read不可。

Artifact URIはopaque referenceでCoreが自動fetchしない。

SQLite/data rootのOS permission/encryption/backup encryptionはdeployment責任。

## 14. MCP exposure

親はtool subsetを選択可能。ただしtool非公開はAuthorizationの代替ではない。

Namespace必須。

## 15. Error information leakage

Provider policyにより `forbidden/not_found` を選択。権限外resource存在を過剰露出しない。

Error detailsもSecretGuard対象。

## 16. 受入条件

1. all public read/write Authorization
2. env/external/human/reusable/conditions Secret拒否
3. per-Attempt materialization
4. missing Secret failure
5. Output exact/substring Secret persistence拒否
6. state.set Secret拒否
7. Artifact URI/metadata Secret拒否
8. Event/error Secret拒否
9. Log Secret redaction
10. transformed Secretは非保証と明記
11. safe YAML
12. path traversal拒否
13. Artifact URI no fetch
14. Runner fencing
