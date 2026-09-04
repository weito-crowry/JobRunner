# 12. Security / Secrets 詳細設計

- Status: Draft v0.6
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `08`, `09`, `11`

## 1. 基本原則

1. Authenticationは親責任。
2. CoreはAuthorizationProvider必須hook。
3. Default AllowAll。
4. 全public read/writeをauthorize。
5. Secret valueをCore管理persistent storageへ平文保存しない。
6. Secret materializeはinternal Action Job `with`だけ。
7. Coreは本格sandbox/arbitrary code標準実行を提供しない。
8. Runnerはinternal fencing。
9. Cross-run Artifact accessもAuthorizationを迂回しない。

## 2. Actor / AccessScope / Authorization

ActorContext:

```text
actor_type
actor_id optional
source optional
claims optional
metadata optional
```

AccessScopeはparent-defined project/workspace/tenant/resource scope。

ChildはParent scopeを継承し権限拡大禁止。

Action Runtime Handleの内部operationもcurrent Workflow RunのActorContext/AccessScopeを引き継ぐ。

## 3. Runner fencing

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id where applicable
```

current ownership不一致のlate update reject。

## 4. SecretsProvider

```python
class SecretsProvider:
    def get(self, name, actor, scope) -> str: ...
```

MVP Secret valueはnon-empty Python `str`。空string/bytes/number/object reject。

Invalid provider value:

```text
category=security
code=secret_value_invalid
retryable=false
```

Binary secret等は将来typed contractで拡張し暗黙変換しない。

Secret name:

```text
^[A-Za-z_][A-Za-z0-9_]*$
```

## 5. Secret reference policy

許可はinternal Action Job `with`のみ。

禁止:

```text
env
External/Human/Reusable with
if/success_if/retry if/continue-on-error
foreach/key/order_by
Workflow outputs/concurrency
```

Persistent Inputはreference marker。各Attempt Action起動直前materialize。Retryはreference固定、rotation後new value許容。

Missing Secret=`secret_not_found`。

## 6. Attempt Secret Set

Runnerはcurrent internal AttemptでmaterializeしたSecretをAttempt専用Setとして保持。

- persistence無し
- Event/debug metadataへ出さない
- Attempt終了時memory reference破棄
- UTF-8 bytesもmanaged file scan用に保持

## 7. Structured SecretGuard

Core管理persistent JSON/textへ書く前にrecursive string scan。

対象:

- Action/Workflow Output
- `state.set`
- Artifact URI/metadata/name
- Event payload
- structured error/details
- idempotency persisted result

Known Secret exact/substring含有 ->

```text
category=security
code=secret_value_persistence_blocked
retryable=false
```

PayloadStore inline/blob前に必須。

## 8. Managed Artifact content guard

`runtime.artifact.put_file` はdurable finalize前にsource fileをstream scan。

- current Attempt known Secret UTF-8 bytes検索
- chunk境界match用overlap
- match時保存拒否/temp copy削除/metadata無し
- binaryでもexact bytes matchならreject

Failure=`secret_value_persistence_blocked`。

### External Reference Artifact

Coreは外部実体内容scan無し。URI/metadataだけSecretGuard。外部実体のSecret管理は親責任。

## 9. Artifact authorization

Canonical ArtifactRefとmaterialize規則は`09`。

### same-run

Current Workflow Run所有ArtifactはRuntime内部resourceとして利用可能。ただしcurrent Actor/AccessScopeのRun accessとArtifact状態を確認する。

### cross-run

別Workflow Run所有Managed Artifactを`runtime.artifact.materialize`する場合:

1. canonical ArtifactRefがcurrent persistent Job Input内に明示存在
2. source Artifact row/dataが存在
3. `AuthorizationProvider.authorize(current_actor, "artifact.read", source_artifact, current_scope)` 相当がallowed
4. source Artifactがcurrent scopeから参照可能

を要求する。

InputにArtifactRefがあることはAuthorizationの代替ではない。

Raw artifact_idだけを知っているcaller/Actionへcross-run materializeを許可しない。

Reusable ChildはParent ActorContext/AccessScopeを継承するため、親からArtifactRefを明示渡ししても権限拡大しない。

External Reference ArtifactはCore materialize無し。

## 10. Execution Log redaction

stdout/stderr/logはknown Secret substringを `[REDACTED]` に置換してwrite。

Redaction前raw lineを別sinkへ保存しない。

## 11. 検出限界

完全検出保証外:

- Base64/hash/encryption
- Secret fragment分割
- current Attemptへmaterializeされていない別機密値

保証はcurrent AttemptでJobRunnerが知るexact substring/UTF-8 bytes。

## 12. Process environment

Parent全environmentをAction childへ無条件継承しない。必要最小非Secret環境 + runtime-only Secret注入。

## 13. YAML / Reusable security

Safe YAML:

- custom tag reject
- duplicate key reject
- merge key reject
- arbitrary include/fetch reject

ReusableはWorkflow root local/registered ID。URL fetch無し。

## 14. Arbitrary code / Sandbox

Coreに `shell:` / arbitrary Python source無し。必要なら親専用Action + Docker等。

Temp directoryはsandboxではない。

## 15. Log / Artifact / filesystem

- Log read authorize
- external path指定read不可
- Managed store key/path Core生成
- put/materialize destination current work_dir内
- External Artifact URI auto-fetch無し
- cross-run Artifact readはexplicit ref + authorization

## 16. MCP exposure / information leakage

Tool subset非公開はAuthorization代替ではない。Namespace必須。

Providerは`forbidden/not_found` policyを選べる。Error detailsもSecretGuard。

## 17. 受入条件

1. all public read/write authorization
2. Secret name/value contract
3. Secret参照位置
4. per-Attempt Secret Set non-persistent
5. Output/State/Event/Error guard
6. PayloadStore spill前guard
7. managed Artifact exact byte guard
8. external Artifact metadata guard/no content scan
9. Log redaction
10. transformed Secret非保証
11. same-run Artifact access
12. cross-run Artifact explicit ref + authorization
13. raw cross-run artifact_id reject
14. Reusable Child no scope escalation
15. Runner fencing
16. safe YAML/path safety
