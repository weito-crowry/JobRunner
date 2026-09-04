# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v0.7
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`

## 1. 目的

CEL/JMESPath、`${{ ... }}`、Input/Job Output/Workflow Output/Artifact/state/Secret参照、Custom Validator連携、condition helperの正規契約を定義する。

## 2. Expression実装

- CEL: `cel-python >=0.5,<0.6`
- JMESPath: `jmespath >=1.1,<2`

独自DSLは作らない。

`jmespath(...)` はCEL custom function bindingとして登録する。親システム任意functionを自由登録する仕組みにはせず、JobRunnerが定義したhelperだけをCEL環境へ公開する。

## 3. 式記法

```yaml
if: ${{ needs.validate.outputs.valid == true }}
with:
  value: ${{ needs.scan.outputs }}
  label: "${{ inputs.symbol }}-${{ inputs.timeframe }}"
```

scalar全体1式は評価結果の型を保持。文字列埋め込みはstring。object/listを暗黙stringifyしない。

## 4. Context

評価場所に応じて:

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

許可外context参照はerror。

## 5. `inputs` / `env`

`inputs`はRun start snapshot。Input fieldの`nullable`規則は`01`。

- missingと明示nullを区別
- nullable=falseのnull reject
- nullable=trueならnull保持
- extra Input reject

`env`はJSON-compatible literal-only。Expression/Secret参照禁止。

## 6. Secrets

`${{ secrets.* }}` はinternal Action Job `with`だけ。

Persistent Inputにはreference markerだけ。値は各Attempt Action起動直前materialize。Secret value contractは`12`のnon-empty string。

## 7. `needs`

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

Group status:

- 未確定/non-terminalあり: `running`
- 全expansion確定 + 全generated Job terminal: `completed`

0件もcompleted。Conclusionは`05`。

## 8. Nested Dynamic parent

`foreach.parent` は暗黙required dependency。Nested condition dependency set:

```text
{foreach.parent} ∪ declared needs
```

同じparentを`needs`へ重複記載しない。

## 9. `state`

read-only expression。`state.set`はRuntime Handle。Job Inputへ解決した値はsnapshot。

## 10. `item` / `iteration`

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

## 11. `failure`

```text
category/code/message/retryable/details
```

Retry condition等で利用。

## 12. `outputs` context

`success_if`内のcurrent Job result。scalar/list/object/nullすべて許可。

例:

```yaml
success_if: ${{ outputs == true }}
success_if: ${{ outputs.failed_count < 3 }}
```

## 13. Workflow Output用 `jobs`

トップレベルWorkflow `outputs`評価時だけ使用。

```text
jobs.<job>.status/conclusion/outputs/artifacts
```

## 14. Job Input構築

最終Job InputはJSON-compatible object。

`$base` optional 1個、object必須。shallow copy + 明示field override。deep merge無し。

Retryはpersistent Input snapshotを再利用。

## 15. Job Output

Action/External Job resultは任意JSON-compatible value:

```text
null / boolean / number / string / array / object
```

Canonical JSON:

- UTF-8
- NaN/Infinity禁止
- deterministic object serialization可能

### 15.1 Result validation pipeline

正規順序:

1. JSON-compatible / canonical JSON validation
2. optional JSON Schema (`outputs.schema`)
3. optional Custom Validator (`validator`)
4. optional `success_if`
5. SecretGuard
6. PayloadStore persistence

Custom Validatorは`01`のValidator Registry callable。Validatorへ渡すInputはpersistent Job Inputで、materialized Secret valueは渡さない。Validatorはresultを変換しない。

### 15.2 Transparent PayloadStore

Canonical JSON bytes:

```text
size <= output-inline-threshold-bytes
  -> SQLite inline
size > threshold
  -> durable filesystem blob
```

Default threshold=4MiB。Validation最大値ではない。

`needs.*.outputs`, `success_if`, Service Output readはstorage kindを意識せず同じJSON valueを得る。

Blob load時はsize/digest検証。欠落/破損はstorage failure。

### 15.3 Attempt history

OutputはAttempt単位immutable。Failed/cancelled AttemptのOutputはcurrentとして公開しない。Current Job Outputはcurrent successful Attemptから解決。

## 16. Workflow Output

トップレベルYAML `outputs` は **name -> expression/literal のmapping** なので、Workflow Output全体のcanonical resultはJSON objectになる。

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
  report: ${{ jobs.report.artifacts.report }}
```

各field値は任意JSON-compatible value。

Workflow Output object自体もPayloadStoreのinline/spill規則を使う。

Reusable WorkflowではこのWorkflow Output objectがParent Job Outputになる。

`outputs: {}` または未指定ならWorkflow Outputは `{}`。

## 17. Artifact reference

ArtifactはOutputとは別のimmutable成果物。Managed ArtifactはArtifactStore、External ReferenceはURI metadata。

## 18. `continue-on-error`

activation時booleanへ評価しsnapshot。Retryで再評価しない。

利用可能: `inputs/needs/env/state/item/iteration/workflow/run/job`。

## 19. Condition helper

Effective success:

- conclusion=`success`
- conclusion=`failure` + dependency `continue-on-error=true`

`skipped`はeffective successではない。

- `success()`: dependency set全件effective success。空ならtrue
- `failure()`: non-allowed failure/blockedあり
- `cancelled()`: Workflow cancelまたはdependency cancelled
- `always()`: dependencies terminalならtrue。ただしWorkflow cancel後のnew activation不可

未指定`if`は`success()`。

通常Job skipped dependency -> downstream default skip。

Dynamic groupは個別skipをaggregateしgroup conclusionがsuccessになり得る。

## 20. `success_if`

JSON Schema + Custom Validator成功後に評価する。

- boolean true -> success
- boolean false -> `success_condition_failed`
- non-boolean -> `expression_type_error`
- expression error -> `expression_evaluation_error`

Payload persistenceはSecretGuard後。

## 21. `order_by` / JMESPath

Order criterionはnon-null stringまたは**finite number**。同criterion内で全candidate同型。

禁止:

```text
bool/object/array/null/NaN/Infinity/混在型
```

`jmespath(value, expression)`はJSON-compatible valueを対象とし、結果型は利用field側で検証する。

## 22. 評価タイミング

| 対象 | 時点 |
| --- | --- |
| concurrency group | Run start前 |
| Dynamic foreach/key/order | expansion |
| if | dependencies terminal後 |
| continue-on-error | activation |
| with | activation/Attempt1前 |
| Secret | 各internal Attempt child起動前 |
| retry if | failed Attempt後 |
| JSON Schema | Job result canonical validation後 |
| Custom Validator | JSON Schema後 |
| success_if | Custom Validator後 |
| Job Payload persistence | success_if/SecretGuard後 |
| Workflow outputs | Workflow success確定直前 |

## 23. 受入条件

1. Job Output scalar/list/object/null
2. Workflow Output always object
3. optional JSON Schema各型
4. Custom Validator execution order/invalid/exception
5. 4MiB inline/spill
6. large Output downstream透過参照
7. blob欠落/破損fail-closed
8. skipped/default condition
9. Nested parent helper
10. Secret利用位置
11. Input nullable/missing/null strictness
12. success_if non-boolean reject
13. order_by NaN/Infinity reject
