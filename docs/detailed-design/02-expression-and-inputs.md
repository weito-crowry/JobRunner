# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `08-persistence.md`, `09-artifacts-logs-state.md`, `12-security-and-secrets.md`

## 1. 目的

CEL/JMESPath、`${{ ... }}`、Input/Output/Artifact/state/Secret参照、persistent Job Input、Custom Validator連携、condition helperの正規契約を定義する。

## 2. Expression実装

- CEL: `cel-python >=0.5,<0.6`
- JMESPath: `jmespath >=1.1,<2`

独自DSLは作らない。

`jmespath(value, expression)` はCEL custom function bindingとしてJobRunnerが登録する。親システム任意functionの自由登録はMVPに含めない。

## 3. 式記法

```yaml
if: ${{ needs.validate.outputs.valid == true }}
with:
  value: ${{ needs.scan.outputs }}
  label: "${{ inputs.symbol }}-${{ inputs.timeframe }}"
```

- scalar全体1式 -> 評価結果の型を保持
- 通常の文字列埋め込み -> string
- object/listを暗黙stringifyしない
- Secretだけは`01`のfull-scalar ruleに従い文字列埋め込み禁止

## 4. Context

Context名:

```text
inputs
needs
env
secrets
state
item
iteration
failure
outputs
workflow
run
job
jobs
```

利用可能contextはfieldごとに固定する。

| Field | Allowed context |
| --- | --- |
| `concurrency.group` | `inputs, env, workflow` |
| Root `foreach` | `inputs, needs, env, state, workflow, run, job` |
| Nested `foreach.items` | Root + `iteration` |
| `key` / `order_by.expr` | `inputs, needs, env, state, item, iteration, workflow, run, job` |
| Job `if` | `inputs, needs, env, state, item, iteration, workflow, run, job` |
| Job `with` | Job `if` context + `secrets` |
| `continue-on-error` | Job `if` context |
| `success_if` | `outputs, inputs, env, item, iteration` |
| `retry.if` | `failure, inputs, env, state, item, iteration, workflow, run, job` |
| Workflow top-level `outputs` | `jobs, inputs, env, state, workflow, run` |

許可外context参照はDefinition validation error。実行時に必要contextがmissingならexpression error。

`success_if` はJob result判定専用なので `needs/state/secrets/failure` を参照させない。

## 5. `inputs` / `env`

`inputs`はWorkflow Run start snapshot。Input fieldの`nullable`規則は`01`。

- missingと明示nullを区別
- nullable=falseのnull reject
- nullable=trueならnull保持
- extra Input reject

`env`はJSON-compatible literal-only。Expression/Secret参照禁止。

## 6. Secret reference

`${{ secrets.NAME }}` はinternal Action Job `with`だけ、かつ1 scalar全体でのみ許可。

Definition evaluatorはSecret valueを解決せず**typed SecretRef**として扱う。

Persistent JSONではSecret valueの代わりにcanonical reference stringを残す。

```text
${{ secrets.API_TOKEN }}
```

同じliteral stringを通常Inputとして受け取る場合と区別するため、別途`secret_bindings`を必ず保存する。

Canonical binding:

```json
{
  "pointer": "/auth/token",
  "name": "API_TOKEN"
}
```

Rules:

- `pointer`=RFC 6901 JSON Pointer
- `name`=`12`のSecret name syntax
- bindingはpointer ASCでsort
- pointer重複禁止
- pointer先のpersistent valueは対応canonical reference stringでなければならない
- Secretをobject keyとして使わない

`secret_bindings=[]`ならSecret無し。

Secret valueは各internal Attempt起動直前にRunnerがmaterializeする。Retryでbinding/nameは固定、value rotationは許容。

## 7. Persistent Job Input

最終Job Inputの論理型はJSON-compatible **object**。

`with`:

- `$base` optional 1個、評価結果object必須
- `$base` shallow copy
- 明示fieldでoverride
- deep merge無し

Activation時に作るpersistent snapshot:

```text
persistent_input      # JSON object; Secret位置はreference string
secret_bindings       # sorted binding array
input_digest          # SHA-256
```

`input_digest`:

```text
SHA-256(
  canonical-json-v1({
    "input": persistent_input,
    "secret_bindings": secret_bindings
  })
)
```

Secret materialized valueはdigestへ入れない。

Internal JobではRunner claimより前にこのsnapshotをDBへ保存する。External/Human/Reusableはactivation時Attempt作成と同時に保存する。Persistence exact columnsは`08`。

Retryは基準failed Attemptの`persistent_input + secret_bindings + input_digest`をexact copyし、`with`を再評価しない。

## 8. Validatorへ渡すInput

Custom Validatorへ渡す`input_data`は**persistent_input**でありexecution inputではない。

したがってSecret-backed fieldにはreference stringが見える。Secret valueは渡さない。ValidatorはSecret fieldの実値を業務判定に使わない。

## 9. `needs`

通常Job:

```text
needs.<job>.status
needs.<job>.conclusion
needs.<job>.outputs
needs.<job>.artifacts
```

`outputs`は任意JSON-compatible value。objectの場合のみfield access可能。

Dynamic group:

```text
needs.<template>.status
needs.<template>.conclusion
needs.<template>.jobs
needs.<template>.outputs
needs.<template>.artifacts
```

- `jobs`: full job_key -> `{status, conclusion, outputs, artifacts}`
- `outputs`: full job_key -> 任意JSON value
- `artifacts`: full job_key -> named Artifact map

Group status/conclusionは`05`。

## 10. Nested Dynamic parent

`foreach.parent` は暗黙required dependency。

Nested condition dependency set:

```text
{foreach.parent} ∪ declared needs
```

同じparentを`needs`へ重複記載しない。

## 11. `state`

Expressionではread-only current Workflow state map。`state.set`はRuntime Handle。

`with`へ解決したstate valueはpersistent Input snapshotへ固定される。

Job `if`/retry等でstateを読む場合はその評価時点のcurrent stateを使う。

## 12. `item` / `iteration`

`iteration.current`:

```text
template_id/key/item/job_key/source_order
```

Nested parent:

```text
template_id/key/item/job_key/source_order
status/conclusion/continue_on_error
outputs/artifacts
```

`outputs`は任意JSON value。

## 13. `failure`

```text
category/code/message/retryable/details
```

Retry condition等で利用。

## 14. `outputs` context / `success_if`

`success_if`内の`outputs`はcurrent Job result。scalar/list/object/nullすべて許可。

```yaml
success_if: ${{ outputs == true }}
success_if: ${{ outputs.failed_count < 3 }}
```

Evaluation:

- boolean true -> success候補
- boolean false -> `success_condition_failed`
- non-boolean -> `expression_type_error`
- evaluation error -> `expression_evaluation_error`

## 15. Workflow Output用 `jobs`

トップレベルWorkflow `outputs`評価時だけ使用。

```text
jobs.<job>.status/conclusion/outputs/artifacts
```

## 16. Job Output

Action/External Job resultは任意JSON-compatible value:

```text
null / boolean / number / string / array / object
```

Validation pipeline:

1. JSON-compatible / canonical-json-v1
2. optional Draft2020-12 `outputs.schema`
3. optional Custom Validator
4. optional `success_if`
5. SecretGuard
6. PayloadStore persistence

### 16.1 Transparent PayloadStore

```text
size <= effective output-inline-threshold-bytes -> SQLite inline
size > threshold -> durable PayloadStore blob
```

Default 4MiB。Validation最大値ではない。

`needs.*.outputs`, `success_if`, Service Output readはstorage kind非依存。

Blob readはexistence/size/digest検証。欠落/破損fail-closed。

### 16.2 Attempt history

OutputはAttempt単位immutable。Failed/cancelled Attempt Outputはcurrent公開しない。Current Job Outputはcurrent successful Attemptから解決。

## 17. Workflow Output

トップレベルYAML `outputs` はname -> expression/literal mappingなので、Workflow Output全体はJSON object。

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
  report: ${{ jobs.report.artifacts.report }}
```

ArtifactRefもfield値として許可。

Workflow Output objectもPayloadStore inline/spill。ReusableではParent Job Outputになる。

未指定/empty -> `{}`。

## 18. ArtifactRef / explicit data flow

ArtifactはOutputとは別のimmutable成果物。Canonical ArtifactRef shapeは`09`。

### same Run

`needs.<job>.artifacts.*` のArtifactRefをJob Inputへmapping可能。

### cross Run / Reusable Child

Coreは別Run Artifactを暗黙探索・自動reuseしない。利用にはArtifactRefをWorkflow/Job InputまたはWorkflow Outputへ明示的に含める。

Managed cross-run materialize条件:

1. ArtifactRefがcurrent persistent Job Input内に存在
2. metadata/data存在
3. current ActorContext/AccessScopeでsource Artifact read許可

External ReferenceはCore materialize対象外。

Persistent Input内ArtifactRefはinput_digestへ固定。Persistent Input外Artifact materializeは`reuse_eligible=false`。

## 19. `continue-on-error`

activation時booleanへ評価しsnapshot。Retryで再評価しない。

## 20. Condition helper

Effective success:

- conclusion=`success`
- conclusion=`failure` + dependency `continue-on-error=true`

`skipped`はeffective successではない。

- `success()`: dependency set全件effective success。空ならtrue
- `failure()`: non-allowed failure/blockedあり
- `cancelled()`: Workflow cancelまたはdependency cancelled
- `always()`: dependencies terminalならtrue。ただしWorkflow cancel後new activation不可

未指定`if`=`success()`。

通常Job skipped dependency -> downstream default skip。Dynamic groupは`05`のaggregate conclusionを使う。

## 21. `order_by` / JMESPath

Order criterionはnon-null stringまたはfinite number。同criterion内で全candidate同型。

禁止:

```text
bool/object/array/null/NaN/Infinity/混在型
```

`jmespath(value, expression)`結果型は利用field側で検証。

## 22. 評価タイミング

| 対象 | 時点 |
| --- | --- |
| concurrency group | Run start前 |
| Dynamic foreach/key/order | expansion |
| Job if | dependencies terminal後 |
| continue-on-error | activation |
| with/persistent Input | activation/Attempt1前 |
| Secret materialize | 各internal Attempt child起動前 |
| retry if | failed Attempt後 |
| JSON Schema | result canonical validation後 |
| Custom Validator | JSON Schema後 |
| success_if | Validator後 |
| Job Payload persistence | success_if/SecretGuard後 |
| Workflow outputs | Workflow success確定直前 |

Manual Retry後のsuccessful descendantは`03/10`に従い、current dependency contextで`if`と**expected persistent Input**を再評価してreuse可否を決める。

## 23. 受入条件

1. field別context allow/deny matrix
2. Secret full-scalar only
3. Secret binding JSON Pointer / sorting / duplicate reject
4. persistent_input + bindings input_digest golden
5. Internal pre-claim Input snapshot persistence
6. Retry exact persistent Input/binding copy
7. Validator receives no Secret value
8. Job Output scalar/list/object/null
9. Workflow Output object
10. optional JSON Schema各型
11. Custom Validator order
12. inline/spill透過
13. skipped/default condition
14. Nested parent helper
15. Input nullable/missing/null strictness
16. success_if context/type restriction
17. order_by NaN/Infinity reject
18. same-run/cross-run ArtifactRef mapping
