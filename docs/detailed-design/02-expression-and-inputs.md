# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`

## 1. 目的

CEL/JMESPath、`${{ ... }}`、Input/Output/Artifact/state/Secret参照とcondition helperの正規契約を定義する。

## 2. Expression実装

- CEL: `cel-python >=0.5,<0.6`
- JMESPath: `jmespath >=1.1,<2`

独自DSLは作らない。

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

`inputs`はRun start snapshot。extra reject、null/missing区別。

`env`はJSON-compatible **literal only**。Expression/Secret参照禁止。

## 6. Secrets

`${{ secrets.* }}` はinternal Action Job `with`だけ。

Persistent Inputにはreference markerだけ。値は各Attempt Action起動直前materialize。

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

### Dynamic group status

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

`success_if`内のcurrent result。**scalar/list/object/nullすべて許可**。

例:

```yaml
success_if: ${{ outputs == true }}
```

```yaml
success_if: ${{ outputs.failed_count < 3 }}
```

後者はobject resultの場合。

## 13. Workflow Output用 `jobs`

Workflowトップレベル`outputs`評価時のみ。

```text
jobs.<job>.status/conclusion/outputs/artifacts
```

## 14. Job Input構築

最終Job InputはJSON-compatible object。

`$base` optional 1個、object必須。shallow copy + 明示field override。deep merge無し。

Retryはpersistent Input snapshotを再利用。

## 15. Job / Workflow Output

Action/External resultとWorkflow OutputはJSON-compatible value。

Canonical JSON条件:

- UTF-8
- NaN/Infinity禁止
- deterministic object serialization可能

Optional JSON Schema validationを適用可能。

### 15.1 Transparent PayloadStore

永続化時にcanonical JSON bytesを計測する。

```text
size <= output-inline-threshold-bytes
  -> SQLite inline JSON
size > threshold
  -> durable filesystem PayloadStore blob
```

default threshold = 4MiB。

**これは保存方式の切替でありvalidation上限ではない。**

`needs.*.outputs`, `jobs.*.outputs`, `success_if`, Service APIのOutput readはstorage kindを意識せず同じJSON valueを取得する。

Payload load時はsize/digestを検証し、blob欠落/破損はstructured storage failure。

### 15.2 Retry / Attempt history

OutputはAttempt単位immutable。Failed/cancelled AttemptのOutputはcurrent Job Outputとして公開しない。最新successful AttemptのOutputをcurrentとする。

## 16. Artifact reference

ArtifactはOutputとは別のimmutable成果物。Reference shapeは`09`。

Managed ArtifactはArtifactStore、External ReferenceはURI metadataとして扱う。

## 17. `continue-on-error`

activation時booleanへ評価しsnapshot。Retryで再評価しない。

利用可能: `inputs/needs/env/state/item/iteration/workflow/run/job`。

## 18. Condition helper

Effective success:

- conclusion=`success`
- conclusion=`failure` + dependency `continue-on-error=true`

`skipped`はeffective successではない。

- `success()`: dependency set全件effective success。空ならtrue
- `failure()`: non-allowed failure/blockedあり
- `cancelled()`: Workflow cancelまたはdependency cancelled
- `always()`: dependencies terminalならtrue。ただしWorkflow cancel後のnew activationは不可

未指定`if`は`success()`。

通常Job skipped dependency -> downstream default skip。

Dynamic groupは個別skipをgroup aggregateしgroup conclusionがsuccessになり得る。

## 19. `success_if`

ResultのJSON Schema検証後に評価。

- true -> success
- false -> `success_condition_failed`
- expression error -> `expression_evaluation_error`

PayloadStoreへの永続化はSecretGuard通過後。`success_if`評価にはActionから返ったin-memory valueを使ってよい。

## 20. `order_by`

criterionはnon-null string/number。同一criterion内同型。bool/object/array/null/混在は禁止。

## 21. JMESPath

`jmespath(value, expression)`。valueはJSON-compatible、expression string。戻り型制約は利用field側。

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
| success_if | result/schema検証後 |
| Payload persistence | success_if/SecretGuard後 |
| Workflow outputs | Workflow success確定直前 |

## 23. 受入条件

1. scalar/list/object/null Output
2. optional JSON Schema各型
3. 4MiB境界inline/spill
4. large Output downstream透過参照
5. blob欠落/破損fail-closed
6. skipped/default condition
7. Nested parent helper
8. Secret利用位置
9. missing/null/type strictness
