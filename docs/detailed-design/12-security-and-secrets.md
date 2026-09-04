# 12. Security / Secrets 詳細設計

- Status: Draft v1.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `08`, `09`, `11`

## 1. 基本原則

1. Authenticationは親責任。
2. CoreはAuthorizationProvider必須hook。
3. Default AllowAll。
4. 全public read/write authorize。
5. Runtime Handleのresource read/writeもAuthorizationを迂回しない。
6. Secret valueをCore persistent storageへ平文保存しない。
7. Secret materializeはinternal Action Job `with`だけ。
8. 本格sandbox/arbitrary code標準実行無し。
9. Runner fencing。
10. Cross-run Artifact accessもAuthorization必須。
11. Provider/RegistryはProcess-local Integration Bootstrapで再構築。

## 2. Actor / AccessScope

ActorContext:

```text
actor_type
actor_id optional
source optional
claims optional
metadata optional
```

AccessScope=parent-defined project/workspace/tenant/resource scope。

ChildはParent scope継承、権限拡大禁止。Runtime Handle内部operationもcurrent Workflow RunのActor/Scopeを引き継ぐ。

ActorContext/AccessScopeはJSON-compatible persistence-safe dataとし、parentはsession token/password等を入れない。Known Secret値はpersist前SecretGuard対象。

## 3. AuthorizationProvider contract

概念contract:

```python
authorize(actor, operation, resource, scope) -> allowed/filtered decision
```

Public Service operationは`11`どおり全件hookを通す。

RunnerServiceがAction Runtime Handle requestを処理する場合も同じActor/Scopeで以下のlogical operationをauthorizeする。

```text
state_get                 -> workflow_state.read
state_set                 -> workflow_state.write
artifact_put_file         -> artifact.create
artifact_register_external-> artifact.create
artifact_materialize      -> artifact.read
```

Denied -> Runtime Handle `runtime_response(ok=false)` で`forbidden`等を返し、resource stateを変更しない。

Action `log/progress/step` messageはcurrent Attempt ownershipでfenceされたtelemetryであり、別resource Service read/writeではないため個別Authorization callは必須にしない。ただしcurrent Attempt ownership不一致なら拒否する。

Default AllowAllでもhook invocation自体を省略しない。

## 4. Integration Bootstrap boundary

Source=`04`。

Parent/RunnerはAuthorizationProvider/SecretsProvider/Registry/StoreをProcess内再構築。Action Runnerへ:

```text
SQLite path/connection
AuthorizationProvider credential
SecretsProvider credential
Store credential
parent authentication/session credential
```

を渡さない。

Action Runnerへ渡るSecretはcurrent Attemptに必要なmaterialized valueだけ。

## 5. Runner fencing

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id
```

Current ownership不一致late update reject。

## 6. SecretsProvider

```python
class SecretsProvider:
    def get(self, name, actor, scope) -> str: ...
```

Secret value=non-empty Python `str`のみ。Empty/bytes/number/object reject=`secret_value_invalid`。

Secret name:

```text
^[A-Za-z_][A-Za-z0-9_]*$
```

Missing=`secret_not_found`。

SecretsProvider自身がparent-specific secret access policyを適用してよい。Core AuthorizationProvider hookとは別責務。

## 7. Secret reference policy

Allowed=internal Action Job `with`のみ。

Forbidden:

```text
env
External/Human/Reusable with
if/success_if/retry if/continue-on-error
foreach/key/order_by
Workflow outputs/concurrency
```

Secret expressionが1 scalar全体を占める場合だけ許可。

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

文字列埋め込みは禁止。加工はAction内部。

## 8. Persistent Secret binding

Secret valueの代わりに`02` persistent Inputへcanonical reference stringを残し、別metadataとしてbindingを保存する。

```json
{
  "pointer": "/token",
  "name": "API_TOKEN"
}
```

- RFC6901 pointer
- pointer unique
- pointer ASC sort
- Secret value無し
- retryでpointer/name固定
- unbound marker-like stringは通常literal

Persistence=`job_runs.pending_secret_bindings_json` / `job_attempts.secret_bindings_json`。

## 9. Attempt Secret materialization

Internal Attempt実行直前:

1. binding list read
2. current Actor/ScopeでSecretsProvider.get
3. value contract検証
4. Attempt Secret Set
5. `04 start`へpersistent_input + bindings + required name->valueだけ送信
6. Action Runnerがmemory上execution input作成

Secret Set:

- persist無し
- debug/Event metadata無し
- Attempt終了時reference破棄
- UTF-8 bytesをManaged Artifact scan用に保持

Retryではbinding固定、Provider value rotation許容。

Custom Validatorはpersistent input/reference stringだけを受ける。

## 10. Structured SecretGuard

Core persistent JSON/textへ書く前にrecursive string scan。

対象:

- Action/Workflow Output
- `state.set`
- Artifact URI/metadata/name
- Event payload
- structured error/details
- idempotency result/adapter metadata
- Actor/AccessScope persistence

Known Secret exact/substring -> `secret_value_persistence_blocked`, retryable=false。

PayloadStore前に必須。

Persistent Secret reference string/nameはSecret valueではないため保存可。

## 11. Managed Artifact content guard

`put_file` durable finalize前にcurrent Attempt known Secret UTF-8 bytesをstream scan。Chunk境界match対応。

Match -> save reject/temp cleanup/metadata無し=`secret_value_persistence_blocked`。

External Reference実体はscan無し。URI/metadataだけGuard。

## 12. Artifact authorization

Same-run Managed Artifactもcurrent Actor/Scope確認。

Cross-run:

1. canonical ArtifactRefがpersistent Job Inputに明示
2. row/data存在
3. source Artifact read Authorization
4. current scopeから参照可能

Raw artifact_idだけでは不可。Reusable ChildはParent Actor/Scope継承。

External ArtifactはCore materialize無し。

`artifact_put_file/register_external`も`artifact.create` Authorizationを通す。

## 13. Execution Log redaction

stdout/stderr/logはknown Secret substringを`[REDACTED]`へ置換してwrite。Raw pre-redaction別sink無し。

All executor共通Log policy=`09`。

## 14. 検出限界

完全検出保証外:

- Base64/hash/encryption
- fragment分割
- current Attemptへmaterializeされていない別機密

保証=current AttemptでCoreが知るexact substring/UTF-8 bytes。

## 15. Process environment

Parent全environmentをAction childへ無条件継承しない。Python/processに必要な最小非Secret環境を基本とし、provider credentialをenvironmentへ一括コピーしない。

## 16. YAML / Reusable security

Safe YAML:

- custom tag/duplicate/merge reject
- arbitrary include/fetch無し
- Reusable URL fetch無し

## 17. Arbitrary code / Sandbox

Coreに`shell:`/arbitrary Python source無し。必要なら親専用Action + Docker等。

Temp directory=sandboxではない。

## 18. MCP / information leakage

Tool非公開はAuthorization代替ではない。Namespace必須。

Providerはforbidden/not_found policyを選べる。Error detailsもSecretGuard。

## 19. 受入条件

1. Bootstrap/provider boundary
2. all public read/write Authorization
3. Runtime Handle state/artifact operations each invoke Authorization hook
4. denied Runtime Handle operation has no side effect
5. telemetry fenced by current Attempt ownership
6. Secret name/non-empty str
7. full-scalar-only placement
8. canonical binding/no value persistence
9. unbound marker literal
10. Retry binding fixed/value rotation
11. Action Runner in-memory materialization
12. Validator no Secret value
13. Structured SecretGuard targets
14. Payload spill pre-guard
15. Managed Artifact byte guard
16. External Artifact no content scan
17. Log redaction
18. transformed Secret non-guarantee
19. same/cross-run Artifact authorization
20. Reusable scope non-escalation
21. Runner fencing/path safety
