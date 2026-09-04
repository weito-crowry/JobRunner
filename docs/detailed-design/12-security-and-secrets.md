# 12. Security / Secrets 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `04`, `08`, `09`, `11`

## 1. 基本原則

1. Authenticationは親責任。
2. CoreはAuthorizationProvider必須hook。
3. Default AllowAll。
4. 全public read/write authorize。
5. Runtime Handleのresource read/writeもAuthorizationを迂回しない。
6. Secret valueをCore persistent storageへ平文保存しない。
7. Secret materializeはinternal Action Job `with`だけ。
8. Secret bindingを持つSuccessful AttemptはResult Reuse不適格。
9. 本格sandbox/arbitrary code標準実行無し。
10. Runner fencing。
11. Cross-run Artifact accessもAuthorization必須。
12. Provider/RegistryはProcess-local Integration Bootstrapで再構築。
13. External Lease ownershipとIdempotency actor isolationは同じcanonical Actor principal identityを使う。

## 2. Actor / AccessScope

ActorContext:

```text
actor_type non-empty string
actor_id optional non-empty string
source optional
claims optional
metadata optional
```

AccessScope=parent-defined project/workspace/tenant/resource scope。

ChildはParent scope継承、権限拡大禁止。Runtime Handle内部operationもcurrent Workflow RunのActor/Scopeを引き継ぐ。

ActorContext/AccessScopeはJSON-compatible persistence-safe dataとし、parentはsession token/password等を入れない。Known Secret値はpersist前SecretGuard対象。

### 2.1 Canonical Actor principal key

Service callerを状態変更・Lease所有者として同一視するcanonical keyを `actor_principal_key` とする。

```text
principal_source = {
  "actor_type": actor.actor_type,
  "actor_id": actor.actor_id,
  "access_scope": access_scope
}

actor_principal_key =
  "apr_" + SHA-256(canonical-json-v1(principal_source)).lowercase_hex64
```

Rules:

- `source/claims/metadata` はprincipal identityへ含めない
- 同じ認証主体がMCP/HTTP/Pythonの別Adapterを通っても、親が同じ`actor_type/actor_id/AccessScope`を与えれば同じprincipalになる
- `actor_id=null`は一般read/writeでは許可できるが、同一主体の復元が必要なExternal Lease ownershipには使わない
- `wf_task_claim` / `wf_task_submit` / `claim_next` は**non-empty `actor_id`必須**。不足時=`claimant_identity_required`
- External Lease `claimant_key` はcurrent Service callerの `actor_principal_key` exact value
- `wf_task_info`でactive `lease_id`を復元できるのはcurrent caller principalとLease `claimant_key`が一致する場合だけ
- Idempotency scopeの`actor_principal_key`もこの定義を使う

このkeyは認証credentialではない。保存・比較用のopaque identityであり、これ単独でAuthorizationを省略しない。

## 3. AuthorizationProvider contract

概念contract:

```python
authorize(actor, operation, resource, scope) -> allowed/filtered decision
```

Public Service operationは`11`どおり全件hookを通す。

RunnerServiceがAction Runtime Handle requestを処理する場合も同じActor/Scopeで:

```text
state_get                  -> workflow_state.read
state_set                  -> workflow_state.write
artifact_put_file          -> artifact.create
artifact_register_external -> artifact.create
artifact_materialize       -> artifact.read
```

をauthorizeする。

Public state inspection API (`wf_state_list/read/history`) は `workflow_state.read`。

Denied -> Runtime Handle `runtime_response(ok=false)` で`forbidden`等を返し、resource stateを変更しない。

Action `log/progress/step`はcurrent Attempt ownershipでfenceされたtelemetryであり個別Authorization callは必須にしない。ただしcurrent Attempt ownership不一致なら拒否する。

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

Non-empty `secret_bindings_json` のSuccessful Attemptは`03`どおり`reuse_eligible=false`。

## 9. Attempt Secret materialization

Internal Attempt実行直前:

1. binding list read
2. unique Secret name集合を作成
3. **各unique nameにつきExactly once** `SecretsProvider.get(name, actor, scope)`
4. value contract検証
5. Attempt Secret Set (`name -> value`) を作成
6. `04 start`へpersistent_input + bindings + exact Secret Setを送信
7. Action Runnerがmemory上execution input作成

同じSecret名が複数pointerで使われてもProviderへ複数回問い合わせない。Attempt中のbinding materialization/log redaction/Artifact scanは全て同じAttempt Secret Setを使う。

Secret Set:

- persist無し
- debug/Event metadata無し
- Attempt終了時reference破棄
- UTF-8 encodeしたbyte sequenceもstream redaction/Managed Artifact scan用に保持

Retryではbinding固定、**新Attempt開始時に再度Exactly once/nameでProvider解決**するためvalue rotationを許容する。

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

`put_file` durable finalize前にcurrent Attempt Secret SetのUTF-8 byte sequenceをstream scanする。

- chunk境界をまたぐmatchを検出する
- 空SecretはそもそもProvider contractで禁止
- 複数Secretが重なる場合もどれか1つがmatchすれば拒否

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

## 13. Execution Log / telemetry redaction

### 13.1 Structured `log` / metadata

Action `log.message`、progress message/unit、Step display name/start metadata/finish metadata等のpersistent telemetry stringはcurrent Attempt Secret Setでredactしてから保存する。

Telemetry表示fieldにKnown Secretが含まれていた場合は`[REDACTED]`へ置換し、Action自体をfailureにはしない。Output/state/Artifact等のbusiness dataは§10どおりfail-closed。

### 13.2 stdout/stderr streaming redactor

stdout/stderrは**raw chunk単位で単純replaceしてはいけない**。Secretがpipe read chunk境界をまたいでも漏れないstreaming redactorを通す。

Canonical requirement:

- Attempt Secret Setの各SecretをUTF-8 bytes化
- raw stdout/stderr bytesを受ける
- matcher stateまたは最大Secret byte長-1以上の未確定suffixを保持
- chunk境界をまたぐSecret byte sequenceを検出
- match部分をUTF-8 `[REDACTED]` bytesへ置換
- redaction後bytesだけをLog decoder/writerへ渡す
- EOF時に未確定suffixをflush
- raw pre-redaction bytesを別sink/fileへ保存しない

UTF-8 decode errorの扱いはExecution Log実装policyに従ってreplacement可能だが、**redactionはdecodeより前のbytes段階**で行う。

## 14. Result ReuseとSecret

Secret valueを保存・hashしないため、後からcurrent Secretと過去成功時Secretの同一性をCoreは判定できない。

したがって:

```text
secret_bindings_json != [] -> reuse_eligible=false
```

Target Job Retry自体は禁止しない。RetryではActionを再実行し、そのAttempt開始時のcurrent Secret valueを使う。

Secretを結果再利用可能にしたい高度な仕組み（Secret version identity/provider generation token等）はMVP外。

## 15. 検出限界

完全検出保証外:

- Base64/hash/encryption
- SecretをAction側で断片へ加工後に別々に出力
- current Attemptへmaterializeされていない別機密

保証=current AttemptでCoreが知るexact Secret string/UTF-8 byte sequence。

## 16. Process environment

Parent全environmentをAction childへ無条件継承しない。Python/processに必要な最小非Secret環境を基本とし、provider credentialをenvironmentへ一括コピーしない。

## 17. YAML / Reusable security

Safe YAML:

- custom tag/duplicate/merge reject
- arbitrary include/fetch無し
- Reusable URL fetch無し
- JSON Schema external `$ref/$dynamicRef` fetch無し。`01`のlocal fragment policyに従う

## 18. Arbitrary code / Sandbox

Coreに`shell:`/arbitrary Python source無し。必要なら親専用Action + Docker等。

Temp directory=sandboxではない。

## 19. MCP / information leakage

Tool非公開はAuthorization代替ではない。Namespace必須。

Providerはforbidden/not_found policyを選べる。Error detailsもSecretGuard。

Public state historyで`include_values=true`を使う場合もAuthorization + response size policy適用。SecretGuard済みstateのみが保存される前提だが、AccessScopeによるread制限を省略しない。

External Task infoのactive Lease IDはAuthorizationだけではなく§2.1 claimant一致も必要。他principalのLease IDをread responseへ露出しない。

## 20. 受入条件

1. Bootstrap/provider boundary
2. Actor principal canonical hash + source exclusion
3. task claim/submit requires non-empty actor_id
4. claimant_key exact principal reuse across adapters
5. other principal cannot recover active lease_id
6. Idempotency uses same actor_principal_key definition
7. all public read/write Authorization
8. public state read uses workflow_state.read
9. Runtime Handle state/artifact operations each invoke Authorization hook
10. denied Runtime Handle operation has no side effect
11. telemetry fenced by current Attempt ownership
12. Secret name/non-empty str
13. full-scalar-only placement
14. canonical binding/no value persistence
15. unique Secret name resolved exactly once per Attempt
16. same Secret multiple bindings use same value
17. Retry re-resolves Secret once/name and can rotate
18. non-empty Secret binding marks reuse ineligible
19. Action Runner in-memory materialization
20. Validator no Secret value
21. Structured SecretGuard targets
22. Payload spill pre-guard
23. Managed Artifact byte guard with chunk-boundary match
24. stdout/stderr byte streaming redaction across chunk boundaries
25. structured log/progress/Step telemetry redaction
26. no raw pre-redaction sink
27. External Artifact no content scan
28. transformed Secret non-guarantee
29. same/cross-run Artifact authorization
30. Reusable scope non-escalation
31. Runner fencing/path safety
