# 12. Security / Secrets 詳細設計

- Status: Draft v0.5
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

MVPのSecret valueは **non-empty Python `str`** に限定する。空string、bytes、number、object等はrejectする。

Invalid provider value:

```text
category=security
code=secret_value_invalid
retryable=false
```

Binary secret等が将来必要なら別typed secret contractとして拡張し、MVPで暗黙変換しない。

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

Missing Secretはchild起動前 `secret_not_found`。

## 6. Attempt Secret Set

Runnerはcurrent internal Attemptでmaterializeしたnon-empty Secret stringをAttempt専用Secret Setとして保持する。

- persistenceしない
- Event/debug metadataへ出さない
- Attempt終了時memory referenceを破棄

各Secret stringをUTF-8 bytesへencodeした値もmanaged file scan用に保持する。

## 7. Structured SecretGuard

Core管理persistent JSON/textへ書く前にrecursive string scan。

対象:

- Action/Workflow Output
- `state.set`
- Artifact URI/metadata/name等のmetadata
- Event payload
- structured error/details
- idempotency persisted result

Known Secret valueがstring全体またはsubstringに含まれたらreject:

```text
category=security
code=secret_value_persistence_blocked
retryable=false
```

PayloadStore inline/blobへ保存する前に必ず通す。

## 8. Managed Artifact content guard

`runtime.artifact.put_file` ではArtifactStoreへdurable finalizeする**前**にsource fileをstream scanする。

- current Attemptのknown Secret UTF-8 byte列を検索
- chunk境界を跨ぐmatchを検出できるoverlapを保持
- match時はmanaged Artifact保存を拒否
- temp copyがあればdelete
- metadata rowを作らない

Failure codeは `secret_value_persistence_blocked`。

Binary fileでもknown Secretの生byte列が存在すればreject。

### External Reference Artifact

Coreは外部実体を読まないため内容scanしない。URI/metadataだけStructured SecretGuard対象。外部実体のSecret管理は親責任。

## 9. Execution Log redaction

stdout/stderr/logは保存拒否ではなくknown Secret substringを `[REDACTED]` に置換してwrite。

Redaction前raw lineを別sinkへ保存しない。

## 10. 検出限界

以下は完全検出保証外:

- Base64等encode
- hash
- encryption
- Secretを複数fragmentへ分割
- current AttemptへSecretsProviderからmaterializeされていない別機密値

保証は「current AttemptでJobRunnerが知っているSecretのexact substring/UTF-8 byte sequence」。

## 11. Process environment

Parent全environmentをAction childへ無条件継承しない。Processに必要な最小非Secret環境 + runtime-only Secret注入。

## 12. YAML / Reusable security

Safe YAML:

- custom tag reject
- duplicate key reject
- merge key reject
- arbitrary include/fetch reject

ReusableはWorkflow root local/registered ID。URL fetch無し。

## 13. Arbitrary code / Sandbox

Coreに `shell:` / arbitrary Python source無し。必要なら親専用Action + Docker等。

Temp directoryはsandboxではない。

## 14. Log / Artifact / filesystem

- Log readはauthorize
- external path指定read不可
- Managed Artifact store key/pathはCore生成
- put/materialize pathはcurrent work_dir内へ限定
- External Artifact URIはCoreがfetchしない

## 15. MCP exposure / information leakage

Tool subset非公開はAuthorization代替ではない。Namespace必須。

Providerは`forbidden/not_found` policyを選べる。Error detailsもSecretGuard対象。

## 16. 受入条件

1. all public read/write authorization
2. Secret name syntax
3. Secret value non-empty str / invalid type reject
4. Secret参照位置
5. per-Attempt Secret Set non-persistent
6. Output/State/Event/Error metadata guard
7. PayloadStore spill前guard
8. managed Artifact text/binary exact byte match reject
9. chunk境界Secret match
10. external Artifact contentはscanしない/metadata guard
11. Log redaction
12. transformed Secret非保証
13. Runner fencing
14. safe YAML/path safety
