# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`

## 1. 目的

CEL / JMESPath、`${{ ... }}`、Input / Output / Artifact / state / Secret 参照と条件 helper の正規契約を定義する。

## 2. 採用実装

Python 3.10+。

- CEL: `cel-python >=0.5,<0.6`
- JMESPath: `jmespath >=1.1,<2`

独自 Expression DSL や親システム任意 callable の CEL 登録は標準機能にしない。

## 3. 式記法

```yaml
if: ${{ needs.validate.outputs.valid == true }}
with:
  count: ${{ needs.scan.outputs.count }}
  label: "${{ inputs.symbol }}-${{ inputs.timeframe }}"
```

scalar 全体が1式なら型保持。文字列埋め込みは string。object/list の暗黙 stringify はしない。

## 4. CEL / JMESPath

CEL:

- Job `if`
- `success_if`
- Retry `if`
- `continue-on-error`
- `foreach / key / order_by`
- `with`
- Workflow concurrency `group`
- Workflow Output mapping

JMESPath は JSON filter/projection 用で、CEL helper `jmespath(value, expression)` としてのみ公開する。

## 5. Context

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

許可されない context の参照は compile/runtime validation error。

## 6. `inputs` / `env`

`inputs` は Run開始時 snapshot。extra Input reject。`null` と `missing` を区別する。

`env` は immutable static values。`${{ secrets.* }}` を `env` 内では使用禁止。

## 7. `secrets`

MVPで `${{ secrets.* }}` を許可するのは **internal Action Job の `with` のみ**。

禁止:

- `env`
- external/human/reusable `with`
- Workflow outputs
- `if`, `success_if`, `continue-on-error`, retry condition
- concurrency
- `foreach/key/order_by`

永続 Job Input には Secret reference marker のみ保存し、値は各 internal Attempt の Action起動直前に materialize する。RetryでSecret値がrotation後の新値になることは許容する。

## 8. `needs`

式から参照する Job/template は `needs` に宣言されていなければならない。

通常 Job:

```text
needs.<job>.status
needs.<job>.conclusion
needs.<job>.outputs
needs.<job>.artifacts
```

Dynamic group:

```text
needs.<template>.status
needs.<template>.conclusion
needs.<template>.jobs
needs.<template>.outputs
needs.<template>.artifacts
```

- `jobs`: full logical job_key -> `{status, conclusion, outputs, artifacts}`
- `outputs`: full logical job_key -> output object
- `artifacts`: full logical job_key -> artifact map

### 8.1 Dynamic group status

- group activation/expansionがまだ完了していない、またはgenerated Jobにnon-terminalがある: `running`
- すべての該当expansionが確定し、generated Jobがすべてterminal: `completed`

0件 expansion も expansion確定後は `completed`。

Group conclusion は `05` に従う。

## 9. Dynamic parent edge と条件 helper

Nested Dynamic template の `foreach.parent` は **暗黙の required dependency** として条件評価に含める。ただし YAML `needs` へ同じ parent を重複記載しない。

Nested templateの condition dependency set は:

```text
{foreach.parent} ∪ declared needs
```

Root Job/template は declared `needs` のみ。

これにより `success()/failure()/cancelled()/always()` は Nested Dynamicでも親generated Jobの状態を含めて一貫して評価する。

## 10. `state`

Workflow Run current state の read-only参照。Expressionから書込禁止。Job Inputへ取り込んだ値は activation 時 snapshot。

## 11. `item` / `iteration`

Dynamic Jobのみ。

`iteration.current`:

```text
template_id
key
item
job_key
source_order
```

`iteration.parent`:

Rootではnull。Nestedでは直近 parent generated Job の:

```text
template_id
key
item
job_key
source_order
status
conclusion
continue_on_error
outputs
artifacts
```

`iteration.ancestors` は outermost -> direct parent。

## 12. `failure`

Retry conditionで利用:

```text
failure.category
failure.code
failure.message
failure.retryable
failure.details
```

## 13. `outputs`

`success_if` のみで current Job Output を参照する。

## 14. Workflow Output 用 `jobs`

トップレベル Workflow `outputs` の評価時だけ使用。

```text
jobs.<job>.status
jobs.<job>.conclusion
jobs.<job>.outputs
jobs.<job>.artifacts
```

Dynamic templateはgroup shape。

## 15. Job Input 構築

`with`をJSON-compatible objectへ解決する。

```yaml
with:
  $base: ${{ needs.prepare.outputs.payload }}
  threshold: 0.8
```

- `$base`: 0または1個、object必須
- shallow copy後、明示fieldで上書き
- deep mergeなし

Secret valueは永続snapshotから除外。

## 16. Output

internal/external resultは JSON-compatible object。

- NaN/Infinity禁止
- optional JSON Schema
- `success_if` は schema validation 後
- canonical UTF-8 JSON byte長が上限超過なら `output_too_large`

## 17. `continue-on-error`

activation時に booleanへ評価し snapshot。Retryでは再評価しない。

利用可能: `inputs/needs/env/state/item/iteration/workflow/run/job`。

`failure/outputs/secrets/jobs`は禁止。

## 18. 条件 helper の正規意味

Job/template `if` は condition dependency set がすべてterminalになった後に評価する。

Dependencyが effective success:

- conclusion=`success`
- conclusion=`failure` かつそのdependencyの snapshotted `continue-on-error=true`

**`skipped` は effective success に含めない。** GitHub Actions寄りに、通常Jobがskipされた場合、そのJobに依存する後段Jobは明示条件がない限り既定 `success()` がfalseとなりskipする。

Helper:

- `success()`: condition dependency set がすべて effective success。空集合ならtrue。
- `failure()`: non-allowed `failure` または `blocked` が1件以上。
- `cancelled()`: Workflow cancel requestあり、または dependencyに`cancelled`あり。
- `always()`: condition dependency set がすべてterminalならtrue。Workflow cancel後の新規開始を許可するものではない。

未指定 `if` は `${{ success() }}`。

### 18.1 default condition false

未指定 `if` の場合:

- cancel由来 -> `cancelled`
- non-allowed failure/blocked -> `blocked`
- upstream `skipped` が原因 -> `skipped`

明示 `if` がfalseなら `skipped`。ただし Workflow cancel 時は `cancelled`。

### 18.2 Dynamic groupとの違い

Dynamic template groupは、個別generated Jobの一部が`skipped`でも `05` のgroup集約規則によりgroup conclusionが`success`になり得る。後段がtemplate groupを`needs`する場合は、その**group conclusion**に対して通常helperを評価する。

## 19. `success_if`

Output validation後:

- true -> success
- false -> `success_condition_failed`
- expression error -> `expression_evaluation_error`

## 20. `order_by`

criterionは non-null string または number。同一criterion内は全candidateで同型。bool/object/array/null/混在型は禁止。

同値なら source order、その後 full logical `job_key`。

## 21. JMESPath helper

`jmespath(value, expression)`:

- expressionはstring
- valueはJSON-compatible
- 戻り値の型制約は利用field側で検証

## 22. 評価タイミング

| 対象 | 時点 |
| --- | --- |
| concurrency group | Workflow Run start前 |
| Dynamic foreach/key/order_by | expansion時 |
| Job/template if | condition dependencies terminal後 |
| continue-on-error | activation時 |
| with | activation / Attempt1開始前 |
| Secret value | 各internal Attempt Action起動直前 |
| Retry if | failed Attempt確定後 |
| success_if | Output validation後 |
| Workflow outputs | Workflow success確定直前 |

## 23. 受入条件

1. Secret利用位置制限
2. env Secret拒否
3. Dynamic group status/conclusion
4. Nested parent edgeがcondition helperへ含まれる
5. parent + declared needs混在
6. skipped dependencyでdefault downstream skip
7. continue-on-error failureでeffective success
8. helper 4種
9. explicit if false
10. missing/null/type strictness
11. full logical job_key参照
12. order_by型検証
