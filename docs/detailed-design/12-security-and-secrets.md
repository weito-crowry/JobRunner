# 12. Security / Secrets 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `08`, `09`, `11`

## 1. 基本原則

1. Authenticationは親責任。
2. CoreはAuthorizationProvider必須hook。
3. Default AllowAll。
4. 全public read/write authorize。
5. Secret valueをCore persistent storageへ平文保存しない。
6. Secret materializeはinternal Action Job `with`だけ。
7. 本格sandbox/arbitrary code標準実行無し。
8. Runner fencing。
9. Cross-run Artifact accessもAuthorization必須。
10. Provider/RegistryはProcess-local Integration Bootstrapで再構築。

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

ChildはParent scope継承、権限拡大禁止。Runtime Handle内部operationもcurrent Actor/Scopeを引き継ぐ。

ActorContext/AccessScopeはJSON-compatible persistence-safe dataとし、parentはsession token/password等を入れない。Known Secret値はpersist前SecretGuard対象。

## 3. Integration Bootstrap boundary

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

## 4. Runner fencing

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id
```

Current ownership不一致late update reject。

## 5. SecretsProvider

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

## 6. Secret reference policy

Allowed=internal Action Job `with`のみ。

Forbidden:

```text
env
External/Human/Reusable with
if/success_if/retry if/continue-on-error
foreach/key/order_by
Workflow outputs/concurrency
```

MVPでは**Secret expressionが1 scalar全体を占める場合だけ**許可。

Allowed:

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

Rejected:

```yaml
with:
  auth: "Bearer ${{ secrets.API_TOKEN }}"
```

加工はAction内部で行う。

## 7. Persistent Secret binding

Secret valueの代わりに`02`のpersistent Inputへcanonical reference stringを残し、別metadataとしてbindingを保存する。

```json
{
  "pointer": "/token",
  "name": "API_TOKEN"
}
```

Properties:

- RFC6901 pointer
- pointer unique
- binding array pointer ASC
- Secret value無し
- retryでpointer/name固定
- unbound marker-like stringは通常literalでありSecretとして解決しない

Persistence=`job_runs.pending_secret_bindings_json` / `job_attempts.secret_bindings_json`。

## 8. Attempt Secret materialization

Internal Attempt実行直前:

1. binding listをread
2. current Actor/ScopeでSecretsProvider.get(name)
3. value contract検証
4. Attempt Secret Set作成
5. `04 start`へpersistent_input + bindings +必要name->valueだけ送信
6. Action Runnerがmemory上execution inputを作る

Secret Set:

- persist無し
- debug/Event metadata無し
- Attempt終了時reference破棄
- UTF-8 bytesをmanaged Artifact scan用に保持

Retryではbinding固定、Provider value rotationは許可。

Custom Validatorはpersistent input/reference stringだけを受け、Secret value無し。

## 9. Structured SecretGuard

Core persistent JSON/textへ書く前にrecursive string scan。

対象:

- Action/Workflow Output
- `state.set`
- Artifact URI/metadata/name
- Event payload
- structured error/details
- idempotency result/adapter metadata
- Actor/AccessScope persistence

Known Secret exact/substring ->

```text
category=security
code=secret_value_persistence_blocked
retryable=false
```

PayloadStore前に必須。

Persistent Secret **reference string/name**はSecret valueではないため保存可。

## 10. Managed Artifact content guard

`put_file` durable finalize前にcurrent Attempt known Secret UTF-8 bytesをstream scan。Chunk境界match対応。

Match -> save reject/temp cleanup/metadata無し=`secret_value_persistence_blocked`。

External Reference実体はscan無し。URI/metadataだけGuard。

## 11. Artifact authorization

Same-run Managed Artifactもcurrent Actor/Scope確認。

Cross-run:

1. canonical ArtifactRefがpersistent Job Inputに明示
2. row/data存在
3. source Artifact read Authorization
4. current scopeから参照可能

Raw artifact_idだけでは不可。Reusable ChildはParent Actor/Scope継承。

External ArtifactはCore materialize無し。

## 12. Execution Log redaction

stdout/stderr/logはknown Secret substringを`[REDACTED]`へ置換してwrite。Raw pre-redaction別sink無し。

All executor共通Log policyは`09`。

## 13. 検出限界

完全検出保証外:

- Base64/hash/encryption
- fragment分割
- current Attemptへmaterializeされていない別機密

保証=current AttemptでCoreが知るexact substring/UTF-8 bytes。

## 14. Process environment

Parent全environmentをAction childへ無条件継承しない。Python/processに必要な最小非Secret環境を基本とし、provider credentialをenvironmentへ一括コピーしない。

## 15. YAML / Reusable security

Safe YAML:

- custom tag/duplicate/merge reject
- arbitrary include/fetch無し
- Reusable URL fetch無し

## 16. Arbitrary code / Sandbox

Coreに`shell:`/arbitrary Python source無し。必要なら親専用Action + Docker等。

Temp directory=sandboxではない。

## 17. MCP / information leakage

Tool非公開はAuthorization代替ではない。Namespace必須。

Providerはforbidden/not_found policyを選べる。Error detailsもSecretGuard。

## 18. 受入条件

1. Bootstrap/provider boundary
2. all public read/write authorization
3. Secret name/non-empty str
4. full-scalar-only placement
5. canonical binding pointer/sort/no value persistence
6. unbound marker literal remains literal
7. Retry binding fixed/value rotation
8. Action Runner materialization only in memory
9. Validator no Secret value
10. Structured SecretGuard targets
11. Payload spill pre-guard
12. Managed Artifact byte guard/chunk boundary
13. External Artifact no content scan
14. Log redaction
15. transformed Secret non-guarantee
16. same/cross-run Artifact authorization
17. Reusable scope non-escalation
18. Runner fencing/path safety
