# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`

## 1. 目的

CEL / JMESPath、`${{ ... }}`、Input / Output / Artifact / state / Secret 参照と条件 helper の正規契約を定義する。

## 2. 採用実装

MVP の Python は 3.10 以上を対象とし、式実装は既存 OSS を使う。

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

- scalar 全体が 1 式なら評価結果の型を保持する。
- 文字列埋め込みでは最終値は string。object/list を暗黙 stringify しない。
- 式を含まない YAML 値はその型のまま扱う。

## 4. CEL / JMESPath の役割

CEL:

- Job `if`
- `success_if`
- Retry `if`
- `continue-on-error`
- `foreach` / `key` / `order_by`
- `with`
- Workflow concurrency `group`
- Workflow/Reusable Output mapping

JMESPath は JSON の filter/projection 用。CEL helper としてのみ公開する。

```yaml
foreach: ${{ jmespath(needs.generate.outputs, 'candidates[?score > `0.8`]') }}
```

## 5. top-level context

評価場所に応じて以下を公開する。

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

存在しない context の参照は validation / resolution error。場所ごとの許可 context を compile 時に検証する。

## 6. `inputs`

Workflow Run 開始時の Input snapshot。immutable。

解決順:

1. caller value
2. schema default
3. optional + default無し = missing
4. required + value無し = start validation failure

`missing` と明示 `null` は区別する。extra Input は reject。

## 7. `env`

Workflow Run 開始時に snapshot した immutable static values。

```yaml
${{ env.MODE }}
```

Secret value を env snapshot に materialize しない。

## 8. `secrets`

Secret Provider から実行直前に materialize する runtime-only 値。

### 8.1 許可位置

MVP では `${{ secrets.* }}` は **internal Action Job の `with` だけ**で許可する。

禁止:

- `external_llm` Job `with`
- `human` Job `with`
- Reusable Workflow 呼び出し Job `with`
- Workflow `outputs`
- concurrency / `if` / `key` / `order_by` / retry condition

Child Workflow が Secret を必要とする場合、Child 自身の internal Action Job が同じ Secret Provider を参照する。

理由は External/Human payload や親子 Input snapshot に Secret 値を混ぜないため。

### 8.2 Snapshot と Retry

永続 Job Input には Secret の参照情報だけを保存し、値は保存しない。Retry は同じ Secret reference を保持するが、**各 Attempt の Action 起動直前に再 materialize**する。

従って親側 Secret が rotation された場合、Retry が新しい Secret 値を使うことは仕様上許容する。「Retry Input固定」は永続 Input と Secret reference が固定という意味であり、runtime-only Secret value の固定を保証しない。

### 8.3 保存禁止

Secret value を以下へ平文保存しない。

- Definition resolved snapshot
- Workflow / Job Input JSON
- Output/Event/Execution Log
- idempotency request hash/result
- IPC debug dump

## 9. `needs`

宣言済み `needs` のみ参照可能。式だけの暗黙 dependency を禁止する。

通常 Job:

```text
needs.<job>.status
needs.<job>.conclusion
needs.<job>.outputs
needs.<job>.artifacts
```

### 9.1 Dynamic Job group

Dynamic template `evaluate` を `needs` に宣言した場合の正規 shape:

```text
needs.evaluate.status
needs.evaluate.conclusion
needs.evaluate.jobs
needs.evaluate.outputs
needs.evaluate.artifacts
```

- `jobs`: generated full logical `job_key` -> `{status, conclusion, outputs, artifacts}`
- `outputs`: generated full logical `job_key` -> output object
- `artifacts`: generated full logical `job_key` -> named Artifact map

個別 generated Job:

```text
needs.evaluate.jobs["evaluate[candidate_a]"].outputs
```

Nested generated Job では full logical key を使用する。

```text
needs.condition.jobs["evaluate[candidate_a]/condition[x]"].outputs
```

raw Dynamic key だけを group map key にしない。異なる親階層で同じ raw key が使われても衝突させないためである。

Dynamic group の status/conclusion 算出は `05-dynamic-jobs.md`。

## 10. `state`

Workflow Run mutable state の current valueをread-only参照する。

```yaml
${{ state.phase }}
```

Expression から書き込み不可。`state.set` は Runtime Handle 経由。Job Input 解決時点の値を永続 Input snapshot に固定する。

## 11. `item` / `iteration`

Dynamic Job でのみ利用可能。

### `item`

現在の Dynamic element。

### `iteration.current`

```text
template_id
key
item
job_key
source_order
```

- `key`: raw key を canonical string 化した値
- `job_key`: full logical key

### `iteration.parent`

Root Dynamic Jobでは `null`。Nested Dynamic Jobでは直近親の snapshot:

```text
template_id
key
item
job_key
source_order
status
conclusion
outputs
artifacts
```

### `iteration.ancestors`

外側から直近親までを順に並べた `iteration.parent` と同じ shape の array。

各 generated Job 作成時に iteration context を snapshot し、Retry で再計算しない。

## 12. `failure`

Retry / failure handling だけで利用可能。

```text
failure.category
failure.code
failure.message
failure.retryable
failure.details
```

failure が無い context で参照したら error。

## 13. `outputs`

`success_if` のみで current Job の検証済み Output を参照する。

```yaml
success_if: ${{ outputs.failed_count < 3 }}
```

## 14. `workflow` / `run` / `job`

read-only metadata。MVPで安定公開する field:

```text
workflow.id
workflow.version
workflow.name
run.id
run.priority
run.started_at
job.id
job.priority
job.attempt
```

Dynamic generated Job の `job.id` は full logical `job_key`。

## 15. Workflow Output 用 `jobs`

トップレベル Workflow `outputs` の評価時だけ使用可能。

```text
jobs.<job>.status
jobs.<job>.conclusion
jobs.<job>.outputs
jobs.<job>.artifacts
```

Dynamic template の場合は `needs` と同じ group shapeを公開する。

Workflow output 以外で `jobs` を参照することは禁止する。通常 Job 間 dependency は `needs` を使う。

## 16. Job Input 構築

`with` を最終 JSON-compatible objectへ解決する。

```yaml
with:
  symbol: ${{ inputs.symbol }}
  candidate: ${{ item }}
```

既存 object 全体を base にする場合:

```yaml
with:
  $base: ${{ needs.prepare.outputs.payload }}
  threshold: 0.8
```

規則:

1. `$base` は 0 または1個。object 必須。
2. base を shallow copy。
3. 同じ `with` の明示 field で上書き。
4. deep merge はしない。
5. `$base` は最終 Action Input には field として残さない。

## 17. Job Input snapshot

Action/External/Human/Reusable の activation 前に、Secret value を除く final Input を canonical JSON として保存する。

含むもの:

- Workflow Input / env
- upstream Output / Artifact ref
- state current value
- Dynamic item/iteration
- literal
- Secret reference marker（internal Action Jobのみ）

Retry はこの snapshot を再利用し、upstream Output/state/item を再評価しない。

## 18. Job Output

Action/external result は JSON-compatible object を基本とする。

- NaN / Infinity 禁止
- optional JSON Schema
- `success_if` より先に JSON / schema 検証
- canonical UTF-8 JSON byte長が `max-job-output-bytes` を超えたら `output_too_large`
- MVP既定 4,194,304 bytes

大きい実体は親/Action側で保存し Artifact reference を返す。

## 19. Artifact reference

```yaml
with:
  source: ${{ needs.export.artifacts.dataset }}
```

式結果は metadata object。実体の読取は Action/親側責任。

## 20. null / missing / 型

- `null`: field は存在し値が null
- `missing`: field/key 自体が無い
- required location の missing は error。暗黙 null 化しない。
- 数字文字列->number、number->string 等の暗黙 conversion はしない。
- boolean context は boolean のみ。truthy/falsy conversion をしない。

## 21. `continue-on-error` 評価

Job activation 時、Job Input と dependency が解決可能になった後、実行前に評価する。

- literal boolean または CEL boolean
- 使用可能: `inputs/needs/env/state/item/iteration/workflow/run/job`
- `failure/outputs/secrets/jobs` は禁止
- 結果を Job Run に snapshotし、Retry で変えない
- expression error は Job activation failure

## 22. condition helper の正規意味

Job `if` は、宣言された `needs` がすべて terminal になった後に評価する。

Dependency 1件を「effective success」とする条件:

- conclusion = `success`、または
- conclusion = `failure` かつ、その dependency の snapshotted `continue-on-error = true`

Helper:

- `success()`: すべての declared need が effective success。`needs` 無しでは true。
- `failure()`: declared need のいずれかが、許容されていない `failure` または `blocked`。Workflow cancel による cancelled は含めない。
- `cancelled()`: Workflow Run に cancel request がある、または declared need のいずれかが `cancelled`。
- `always()`: declared need がすべて terminal なら true。`needs` 無しでは true。ただし Workflow Run cancel 後に新 Job を開始する許可にはならない。

Job `if` 未指定は `${{ success() }}`。

### default condition が false の終端

- 原因が upstream `skipped` のみ: current Job = `completed/skipped`
- upstream に許容されていない `failure` / `blocked` がある: `completed/blocked`
- Workflow cancel: `completed/cancelled`
- 明示 `if` が false: 原因に関係なく `completed/skipped`。ただし Workflow cancel 時は `cancelled`

## 23. `success_if`

Action/external resultが正常終了し、Output JSON/schema validation 後に評価。

- true -> success
- false -> failure (`success_condition_failed`)
- expression error -> failure (`expression_evaluation_error`)

`success_if` 未指定なら正常終了 + Output validation success で success。

## 24. `order_by` の値

各 criterion は `null` ではない **string または number** を返す必要がある。1回の expansion 内で同じ criterion の全 candidate は同じ型でなければならない。

禁止: bool/object/array/null/混在型。

`direction` は `asc | desc`。全criterion同値なら source order、その後 full logical `job_key` で deterministic tie-break。

## 25. JMESPath helper

`jmespath(value, expression)`:

- expression は string
- value は JSON-compatible
- compile/evaluation error は structured expression failure
- 戻り値の型制約は利用 field 側で検証する（例 `foreach` は array）

## 26. Expression error

代表 code:

```text
expression_compile_error
expression_context_error
expression_missing_value
expression_type_error
expression_evaluation_error
jmespath_compile_error
jmespath_evaluation_error
output_too_large
```

Runtimeでしか分からない missing upstream field/state/Secret 等は、その Job の activation/Input resolution failure とする。

## 27. 評価タイミング一覧

| 対象 | 時点 |
| --- | --- |
| Workflow concurrency group | Workflow Run start前 |
| Dynamic `foreach/key/order_by` | expansion時 |
| Job `if` | declared needs terminal後 |
| `continue-on-error` | Job activation時 |
| Job `with` | activation/Attempt 1開始前 |
| Secret value | 各 internal Attempt Action起動直前 |
| Retry `if` | failed Attempt確定後 |
| `success_if` | Output validation後 |
| Workflow `outputs` | Workflow success確定直前 |

## 28. 受入条件

1. CEL/JMESPath compile/evaluate
2. full scalar型保持/文字列埋め込み
3. missing/null/type strictness
4. declared `needs` 外参照拒否
5. exact Dynamic group/individual reference
6. exact iteration parent/ancestors
7. `jobs` context は Workflow outputs限定
8. helper 4種の success/failure/skipped/cancelled/continue-on-error 組合せ
9. `continue-on-error` snapshot
10. Secret internal-only / external-human-reusable拒否
11. Retry Secret再materialize
12. `$base` shallow override
13. Output schema / 4 MiB limit
14. `order_by` 型検証
