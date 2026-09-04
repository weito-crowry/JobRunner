# 12. Security / Secrets 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `04-runner-and-ipc.md`, `08-persistence.md`, `11-service-api-and-mcp.md`

## 1. 目的

認証/認可境界、ActorContext/AccessScope、Runner trust boundary、Secrets注入/非永続化、Log/Artifact/Tempの安全境界を定義する。

## 2. 基本原則

1. Authenticationは親システム責任。
2. CoreはAuthorizationProviderを必須hookとして持つ。
3. MVP defaultはAllowAll。
4. **全public Service read/write operationがAuthorizationProviderを通る。**
5. Secret valueをYAML/SQLite/Event/Execution Logへ平文保存しない。
6. Secretはinternal Action Jobだけへ必要時materialize。
7. Coreは本格sandbox/任意code executionを標準提供しない。
8. Runner内部identityはexternal user authとは別のfencingで守る。

## 3. Trust boundary

```text
External caller
 -> Parent authentication
 -> ActorContext/AccessScope
 -> Adapter
 -> AuthorizationProvider
 -> Public Runtime Service
 -> Persistence

Parent Supervisor
 -> Runner Process identity/fencing
 -> Internal RunnerService
 -> Persistence

Runner
 -> Common Action Runner child
 -> registered Action
```

Action Runner childはDBへ直接接続しない。

## 4. ActorContext

```text
actor_type
actor_id optional
source optional
claims optional
metadata optional
```

例 `user/service/agent/system`。Coreが不要属性を自動収集しない。

## 5. AccessScope

抽象scope:

```text
project_id optional
workspace_id optional
tenant_id optional
resource_tags optional
```

意味は親側が定義。CoreはscopeをResource query/filterへ渡す。

## 6. AuthorizationProvider

```python
class AuthorizationProvider:
    def authorize(self, actor, operation, resource, scope) -> AuthorizationDecision:
        ...
```

Decision:

```text
allowed
reason optional
filtered_scope optional
```

AllowAllでも呼出経路を省略しない。

## 7. Authorization対象

Read含む全public operation:

```text
Workflow Definition list/info
Workflow Run list/info/start
pause/resume/cancel/retry/priority
External task info/claim/submit
Human review list/info/submit
Artifact info/history
Log read
Runner info
public state read/writeが追加された場合それも
```

Action Runtime Handleのstate/artifact操作はcurrent Workflow RunのActor/AccessScopeを引き継ぐ内部operation。

## 8. Child Workflow権限

Child RunはParent ActorContext/AccessScopeを継承する。権限拡大禁止。Child側で狭めることは可能。

Childのpublic readは通常Authorization。Public controlは`06/11`によりMVP拒否。

## 9. Runner identity / fencing

Runner internal update:

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id where applicable
```

をcurrent DB ownershipと照合する。旧Runtime/旧Runner instanceのlate update拒否。

Runner internal Serviceは外部へ公開しない。DB path/data rootはRunner起動configとして親Supervisorから渡せるが、Action Runner childへ渡さない。

Parent lifecycle pipe EOFでもRunner終了を要求する。

## 10. SecretsProvider

```python
class SecretsProvider:
    def get(self, name, actor, scope):
        ...
```

親側dict/env/secret manager adapterでよい。

## 11. Secret reference policy

```yaml
jobs:
  call_api:
    action: api.call
    with:
      token: ${{ secrets.API_TOKEN }}
```

`${{ secrets.* }}`はinternal Action Job `with`だけで許可。

禁止:

```text
External LLM with
Human with
Reusable Workflow parent with
if / retry if / continue-on-error / foreach / key / order_by
Workflow outputs / concurrency
```

Child Workflowは自分のinternal ActionからSecretを参照する。

## 12. Materialization / Retry

Persistent Job InputにはSecret reference markerだけ保存。値は各AttemptのAction起動直前にProviderから解決する。

同じreferenceをRetryで維持するが、rotation後の新value使用を許容する。

missing SecretはAction childを起動する前に`secret_not_found` validation/resolution failure。

## 13. Secret allow set

Action registration metadataで必要Secret名を宣言可能:

```python
registry.register(
    "llm.call",
    version="1",
    secrets=["OPENAI_API_KEY"],
    callable=call_llm,
)
```

MVPではallow-list必須ではないが、親policyが許可Secret名を制限できるhookを持つ。

YAMLが参照するSecret名syntaxは:

```text
^[A-Za-z_][A-Za-z0-9_]*$
```

## 14. 非永続化

Secret valueを以下に保存しない:

- Definition resolved JSON
- Workflow/Job Input JSON
- Workflow/Job Output
- Event payload
- Execution Log
- Idempotency request hash/result
- IPC debug dump
- Runner metadata

Action Runnerへのstart IPCにはruntime-only fieldとして渡せる。

## 15. Redaction

Known Secret valueをExecution Log write pipeline入口でredactできるhookを持つ。

完全保証ではない。ActionがSecretを変形/分割して出した場合の完全検出はCore責任外。

Redaction前のlog lineを別debug sinkへ平文保存しない。

## 16. Process environment

親Process全environmentをAction Runnerへ無条件継承しない。

Action child environmentはProcess起動に必要な最小値 + 親が明示した非Secret環境を基本とし、SecretはIPC/専用注入で渡す。

## 17. Arbitrary code / Sandbox

Coreに:

```text
shell: arbitrary command
python: arbitrary source
```

を設けない。

必要なら親が専用Actionを登録し、そのAction内でDocker等を使う。

Attempt temp directoryは整理目的でありsandboxではない。ActionはOS権限上他pathへアクセスできる可能性がある。

## 18. YAML security

Safe loaderを使用し:

- custom/object tags拒否
- duplicate mapping key拒否
- merge key `<<`拒否
- arbitrary include/fetch拒否

Reusable `uses`もlocal root/registered IDだけでURL fetch無し。

## 19. Execution Log

Log readはAuthorization対象。External callerへfilesystem pathをread APIとして受け付けない。

Serviceはattempt_idから内部relative pathを解決しdata root外へ出ないことを確認する。

## 20. Artifact URI

Artifact URIはopaque reference。Coreは自動fetchしないためCore Artifact subsystemがSSRF clientにならない。

URI interpretation/authorizationは実体を扱うAction/親側責任。

## 21. SQLite / filesystem

SQLite/data rootのOS permission、disk encryption、backup encryptionはdeployment/親責任。CoreはMVP標準DB encryptionを提供しない。

DBへSecretを入れないことを第一防御とする。

## 22. MCP exposure

親はtool subsetを選べる。Read-only用途ならstate-changing toolを登録しない。

Namespace必須により複数親システムの同名操作混同を減らす。

Tool非公開はAuthorizationの代替ではない。公開toolもAuthorizationを通る。

## 23. Error information leakage

AuthorizationProviderは`forbidden`と`not_found`の使い分けをpolicyとして選べる。権限のないcallerへ他tenant/resourceの存在を必要以上に露出しない。

Error details/EventへSecret値を含めない。

## 24. Audit

State-changing operationはactor/source/request_idをEventへ記録可能。Read Eventを全件auditすることはMVP必須ではないがProvider/Adapter hookを追加可能。

## 25. 将来拡張

```text
RBAC/ABAC
PostgreSQL RLS
Secret Manager adapter
Docker/OS sandbox Action
signed external claim
remote Runner authentication
```

MVP Coreを依存させない。

## 26. 受入条件

1. 全public read/write Authorization
2. AllowAll/Deny/filtered scope
3. Child scope非拡大/control拒否
4. Runner internal fencing
5. Action child DB非接続
6. Secret internal Action withのみ
7. external/human/reusable/conditions Secret拒否
8. reference persistence + per-Attempt materialize
9. missing Secret child起動前failure
10. Secret DB/Event/Log/Idempotency不在
11. redaction hook
12. safe YAML duplicate/tag/merge拒否
13. Log path traversal拒否
14. Artifact URI no fetch
15. MCP subset + Authorization
