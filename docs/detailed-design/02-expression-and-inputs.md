# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v1.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `08`, `09`, `12`

## 1. 目的

CEL/JMESPath、`${{ ... }}`、Input/Output/Artifact/state/Secret参照、persistent Job Input、Custom Validator連携、condition helper、各Expression contextの正規契約を定義する。

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

## 4. Context一覧と公開shape

MVP context名:

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

Core提供contextのshapeを以下へ固定する。記載のないfieldを暗黙追加しない。

### 4.1 `workflow`

```json
{
  "id": "workflow-id",
  "name": "Example Workflow",
  "version": 1
}
```

Definition snapshot由来でRun中不変。

### 4.2 `run`

```json
{
  "id": "wfr_..."
}
```

MVPではpriority/status/run_attempt/timestamp等をExpression contextへ公開しない。Scheduling変更を業務条件へ暗黙混入させないため。

### 4.3 `job`

```json
{
  "template_id": "evaluate",
  "key": "evaluate[a]",
  "executor": "internal"
}
```

- static concrete Job: `template_id=static job id`, `key=job_key`
- generated concrete Job: `template_id=dynamic template id`, `key=full job_key`
- Dynamic template expansion中: `template_id=template id`, `key=null`, `executor=resolved template executor`

Job status/priority/Attempt番号はMVP contextへ公開しない。

### 4.4 `inputs`

Workflow Run start時のWorkflow Input snapshot。

### 4.5 `env`

Workflow Definitionのliteral-only immutable map。

### 4.6 `state`

Current Workflow stateのread-only map。存在しないnameは**missing**でありnullへ暗黙変換しない。

### 4.7 Unknown/missing field

存在しないcontext field/pathを読む式はfieldごとの通常CEL evaluation error。Coreがnull/defaultへ補完しない。

## 5. Field別Allowed Context

| Field | Allowed context |
| --- | --- |
| `concurrency.group` | `inputs, env, workflow` |
| Root `foreach` | `inputs, needs, env, state, workflow, run, job` |
| Nested `foreach.items` | Root + `iteration` |
| `key` / `order_by.expr` | `inputs, needs, env, state, item, iteration, workflow, run, job` |
| Job `if` | `inputs, needs, env, state, item, iteration, workflow, run, job` |
| Job `with` | Job `if` context + `secrets` |
| `continue-on-error` | Job `if` context |
| `success_if` | `outputs, inputs, env, item, iteration, workflow, run, job` |
| `retry.if` | `failure, inputs, env, state, item, iteration, workflow, run, job` |
| Workflow top-level `outputs` | `jobs, inputs, env, state, workflow, run` |

許可外context参照はDefinition validation error。必要contextが実行時missingならexpression error。

`success_if` はJob result判定専用なので `needs/state/secrets/failure` を参照させない。

## 6. Secret reference

`${{ secrets.NAME }}` はinternal Action Job `with`だけ、かつ1 scalar全体でのみ許可。

Definition evaluatorはSecret valueを解決せずtyped SecretRefとして扱う。

Persistent JSONではSecret valueの代わりにcanonical reference stringを残す。

```text
${{ secrets.API_TOKEN }}
```

同じliteral stringを通常Inputとして受け取る場合と区別するため、別途`secret_bindings`を保存する。

Canonical binding:

```json
{
  "pointer": "/auth/token",
  "name": "API_TOKEN"
}
```

Rules:

- `pointer`=RFC 6901 JSON Pointer
- `name`=`12` Secret name syntax
- pointer ASC sort
- pointer重複禁止
- pointer先persistent valueは対応canonical reference string
- Secretをobject keyとして使わない

`secret_bindings=[]`ならSecret無し。

Secret valueは各internal Attempt起動直前materialize。Retryでbinding/name固定、value rotation許容。

**Secret bindingが1件でも存在するAttemptは自動Result Reuse不可**とする。Secret materialized valueを永続化・digest化しないため、過去Attemptと現在のSecret値が同一であることをCoreが証明できないため。RetryではActionを再実行し、各Attempt開始時のcurrent Secret valueを使う。

## 7. Persistent Job Input

最終Job Input論理型=JSON-compatible **object**。

`with`:

- `$base` optional 1個、評価結果object必須
- shallow copy
- 明示field override
- deep merge無し

Activation時snapshot:

```text
persistent_input
secret_bindings
input_digest
```

`input_digest`:

```text
SHA-256(canonical-json-v1({
  "input": persistent_input,
  "secret_bindings": secret_bindings
}))
```

Secret materialized valueはdigestへ入れない。したがってdigest一致だけではSecret付きAttemptのReuse可否を満たさない。`secret_bindings != []`なら`03/08`どおり`reuse_eligible=false`。

InternalはRunner claim前にpending snapshotをDB保存。External/Human/Reusable初回はAttempt作成と同時。Retryはfailed Attempt snapshotをexact copyし`with`を再評価しない。

## 8. Validatorへ渡すInput

Custom Validatorの`input_data`=persistent_input。Execution input/Secret valueは渡さない。

## 9. `needs`

通常Job:

```text
needs.<job>.status
needs.<job>.conclusion
needs.<job>.outputs
needs.<job>.artifacts
```

`outputs`は任意JSON value。objectの場合のみfield access可能。

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

```text
condition dependency set = {foreach.parent} ∪ declared needs
```

同じparentを`needs`へ重複記載しない。

## 11. `item` / `iteration`

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

## 12. `failure`

```text
category/code/message/retryable/details
```

Retry condition等で利用。

## 13. `outputs` context / `success_if`

`outputs`=current Job result。scalar/list/object/null可。

```yaml
success_if: ${{ outputs == true }}
success_if: ${{ outputs.failed_count < 3 }}
```

- true -> success候補
- false -> `success_condition_failed`
- non-boolean -> `expression_type_error`
- evaluation error -> `expression_evaluation_error`

## 14. Workflow Output用 `jobs`

トップレベルWorkflow `outputs`評価時だけ使用。

```text
jobs.<job>.status
jobs.<job>.conclusion
jobs.<job>.outputs
jobs.<job>.artifacts
```

Dynamic templateは`job`名にgroup aggregateを返す。Static/concrete Jobはcurrent stateを返す。

## 15. Job Output

Internal/External resultは任意JSON-compatible value:

```text
null / boolean / number / string / array / object
```

Validation pipelineは`01`。

Transparent PayloadStore:

```text
size <= effective threshold -> SQLite inline
size > threshold -> durable blob
```

Default 4MiB。Maxではない。

Attempt Outputはimmutable。Current Job Output=current successful Attempt。

Human/Reusable Output semanticsは`01/06/07`。

## 16. Workflow Output

トップレベル`outputs`=name -> expression/literal mappingなので全体JSON object。

ArtifactRefもfield値可。PayloadStore inline/spill。ReusableではParent Job Outputになる。

未指定/empty=`{}`。

## 17. ArtifactRef / explicit data flow

Canonical ArtifactRef=`09`。

Same Runは`needs.<job>.artifacts.*`からInput mapping可。

Cross-run/Reusable ChildではArtifactRefをWorkflow/Job InputまたはWorkflow Outputへ明示的に含める。暗黙探索/自動reuse無し。

Managed cross-run materialize条件:

1. ArtifactRefがcurrent persistent Job Input内
2. metadata/data存在
3. current Actor/Scopeでsource Artifact read許可

Persistent Input内ArtifactRefはinput digestへ固定。Persistent Input外materializeはreuse ineligible。

## 18. `continue-on-error`

Activation時booleanへ評価しsnapshot。Retryで再評価しない。

## 19. Condition helper

Effective success:

- conclusion=success
- conclusion=failure + dependency continue-on-error=true

`skipped`はeffective successではない。

- `success()`: dependency set全件effective success。空ならtrue
- `failure()`: non-allowed failure/blockedあり
- `cancelled()`: Workflow cancelまたはdependency cancelled
- `always()`: dependencies terminalならtrue。ただしWorkflow cancel後new activation不可

未指定`if`=`success()`。

通常Job skipped dependency -> downstream default skip。Dynamic groupは`05` aggregate conclusion。

## 20. `order_by` / JMESPath

Order criterion=non-null stringまたはfinite number。同criterion内全candidate同型。

禁止=`bool/object/array/null/NaN/Infinity/混在型`。

`jmespath(value, expression)`結果型は利用field側で検証。

## 21. 評価タイミング

| 対象 | 時点 |
| --- | --- |
| concurrency group | Root/Child Run start前 |
| Dynamic foreach/key/order | expansion |
| Job if | dependencies terminal後 |
| continue-on-error | activation |
| with/persistent Input | activation/Attempt1前 |
| Secret materialize | internal Attempt child起動前 |
| retry if | failed Attempt後 |
| JSON Schema | result canonical validation後 |
| Custom Validator | JSON Schema後 |
| success_if | Validator後 |
| Job Payload persistence | validation/SecretGuard後 |
| Workflow outputs | Workflow success確定直前 |

Manual Retry後successful descendantは`03/10`どおりcurrent contextで`if`とexpected persistent Inputを再評価してreuse可否を決める。ただしSecret binding付きAttemptは一致判定以前にReuse不適格。

## 22. 受入条件

1. workflow/run/job exact shape + unknown field error
2. Dynamic template job.key=null
3. field別context allow/deny matrix
4. state missing != null
5. Secret full-scalar only
6. Secret binding JSON Pointer/sort/duplicate reject
7. input_digest golden
8. Secret binding does not prove value identity / reuse_eligible=false
9. Internal pre-claim snapshot
10. Retry exact binding copy
11. Validator no Secret value
12. Job Output arbitrary JSON
13. Workflow Output object
14. Schema/Validator/success_if order
15. inline/spill
16. skipped/default condition
17. Nested parent helper
18. Input nullable/missing/null
19. order_by type rejection
20. same/cross-run ArtifactRef
