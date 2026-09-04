# 02. Expression / Inputs / Outputs 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `docs/detailed-design/01-workflow-definition.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner における式評価、Workflow Input、Job Input、Job Output、Workflow 共有値、Artifact 参照の解決規則を定義する。

対象:

- `${{ ... }}` 式記法
- CEL の利用範囲
- JMESPath の利用範囲
- 式の評価 context
- Workflow Input
- `env`
- Secrets 参照
- `needs` 参照
- Dynamic Job の `item` 参照
- Workflow state 参照
- Artifact 参照
- Job Input の構築順序
- Job Output の保存・検証
- null / missing / type mismatch の扱い
- 静的検証と実行時検証

Dynamic Job の展開アルゴリズムそのものは `05-dynamic-jobs.md`、Artifact の永続管理は `09-artifacts-logs-state.md` で定義する。

## 2. 基本原則

1. JobRunner 独自の Expression DSL は作らない。
2. 条件・簡単な値計算には CEL を使用する。
3. 複雑な JSON 抽出・検索には JMESPath を使用する。
4. YAML 上の式は GitHub Actions 風の `${{ ... }}` で表現する。
5. Input / Output / Artifact / state の参照はすべて明示的に行う。
6. 存在しない値を推測して補完しない。
7. 型不一致や参照不正は fail-closed とする。
8. Workflow Run 開始時 Input は snapshot し、Run 中に書き換えない。
9. Retry 時は元 Job Input を再利用し、Input を変更しない。
10. Secrets は式から参照可能だが、永続化・通常ログ出力しない。

## 3. 式記法

### 3.1 基本形

式は `${{ ... }}` で囲む。

```yaml
if: ${{ needs.validate.outputs.valid == true }}
```

```yaml
with:
  symbol: ${{ inputs.symbol }}
```

### 3.2 YAML scalar 全体が式の場合

値全体が 1 個の式である場合、評価結果の型をそのまま保持する。

```yaml
with:
  count: ${{ needs.scan.outputs.count }}
```

`count` が integer なら Job Input にも integer として渡す。

### 3.3 文字列への埋め込み

文字列中への式埋め込みも許可する。

```yaml
with:
  label: "${{ inputs.symbol }}-${{ inputs.timeframe }}"
```

この場合、最終結果は string とする。

オブジェクト・配列を文字列へ暗黙変換することは禁止する。

必要な場合は明示 helper を使用する。

### 3.4 式を含まない値

通常 YAML 値としてそのまま扱う。

```yaml
with:
  mode: research
  limit: 100
  enabled: true
```

## 4. Expression Engine

### 4.1 CEL

CEL は以下に使用する。

- `if`
- `success_if`
- Retry `if`
- `foreach`
- Dynamic Job `key`
- Dynamic Job `order_by`
- `with` の値解決
- Workflow concurrency `group`
- Reusable Workflow input / output mapping
- 単純な数値・文字列・boolean 計算

### 4.2 JMESPath

JMESPath は JSON / map / list からの複雑な抽出・filter・projection に使用する。

JMESPath 自体を YAML の別 DSL として増やすのではなく、CEL context に `jmespath(value, expression)` helper を公開する。

例:

```yaml
foreach: ${{ jmespath(needs.generate.outputs, 'candidates[?score > `0.8`]') }}
```

戻り値は JMESPath の評価結果を JSON-compatible value として CEL 側へ戻す。

### 4.3 任意関数の登録

親システムが CEL へ任意 Python callable を直接登録する仕組みは MVP の標準機能にしない。

理由:

- Workflow YAML の可搬性を下げる
- 静的検証が難しくなる
- 任意コード実行境界が曖昧になる

ドメイン固有ロジックは原則 Action / Validator に置く。

## 5. Expression Context

式評価時に使用可能な top-level context を以下とする。

```text
inputs
needs
env
secrets
state
item
iteration
failure
workflow
job
run
```

context は評価場所に応じて一部のみ有効になる。

存在しない context を参照した式は validation error とする。

## 6. `inputs`

Workflow Run 開始時に確定した Workflow Input snapshot を参照する。

```yaml
${{ inputs.symbol }}
```

Input は Workflow Run 中 immutable。

### 6.1 Workflow Input 型

MVP で直接扱う基本型:

```text
string
integer
number
boolean
object
array
null
```

Workflow Input schema は JSON Schema 互換の型定義を基本とする。

### 6.2 required / default

例:

```yaml
inputs:
  symbol:
    type: string
    required: true

  timeframe:
    type: string
    required: false
    default: M5
```

解決順序:

1. caller supplied value
2. schema default
3. required=false かつ default なしなら missing
4. required=true かつ値なしなら Workflow Run を開始しない

`missing` と明示 `null` は区別する。

### 6.3 extra input

Workflow 定義に存在しない Input を caller が渡した場合、既定では reject する。

未知 Input を黙って無視しない。

## 7. `env`

Workflow YAML に定義する Workflow Run 内の静的値。

```yaml
env:
  MODE: research
```

参照:

```yaml
${{ env.MODE }}
```

`env` は Workflow Run 中 immutable。

### 7.1 env の用途

- 固定設定
- Workflow 内で複数 Job が共有する定数
- YAML の重複削減

Secrets を `env` に値として展開・snapshot して永続保存してはならない。

## 8. `secrets`

親システムが Runtime へ提供する Secret Provider から実行時に解決する。

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

### 8.1 永続化禁止

Secret value は以下へ平文保存しない。

- Workflow Definition snapshot の展開後値
- Workflow Input snapshot
- Job Input snapshot の通常 JSON
- SQLite Event payload
- Execution Log
- Debug log
- Idempotency response body

JobRunner が Job Input snapshot を保存する際、Secret を含む field は secret reference / redacted marker と実行時 materialized value を分離して扱う。

実Actionへ渡す直前に Secret value を materialize する。

### 8.2 Secret missing

必要 Secret が存在しない場合、Job を開始せず validation / resolution failure とする。

## 9. `needs`

上流 Job の結果を参照する。

基本形:

```yaml
${{ needs.build.outputs.package }}
```

### 9.1 参照可能条件

`needs.<job>` を参照する Job は、原則その Job を `needs` に宣言していなければならない。

```yaml
analyze:
  needs: [build]
```

依存関係にない Job を式だけで参照することは禁止する。

理由:

- DAG とデータ依存を一致させる
- 実行順序の暗黙依存を作らない

### 9.2 `needs.<job>` の基本形

少なくとも以下を公開する。

```text
needs.<job>.status
needs.<job>.conclusion
needs.<job>.outputs
needs.<job>.artifacts
```

例:

```yaml
if: ${{ needs.validate.conclusion == 'success' }}
```

```yaml
with:
  report: ${{ needs.validate.outputs.report }}
```

### 9.3 Dynamic Job 群

Dynamic Job template を `needs` 参照した場合は、生成された Job 群を stable key 付き map として扱う。

概念例:

```json
{
  "candidate_a": {
    "status": "completed",
    "conclusion": "success",
    "outputs": {"score": 0.91}
  },
  "candidate_b": {
    "status": "completed",
    "conclusion": "success",
    "outputs": {"score": 0.82}
  }
}
```

詳細は Dynamic Job 詳細設計で定義する。

## 10. `state`

Workflow Run 内の mutable shared state の current value を読む。

```yaml
${{ state.current_phase }}
```

YAML Expression から state を変更することは禁止する。

変更は Action Runtime Handle の `state.set(...)` を通す。

式は read-only。

### 10.1 評価時点

`state` は式を評価するその時点の current value を参照する。

Job Input は Attempt 開始時に解決・snapshot するため、実行開始後に state が変わってもその Attempt の Input は変化しない。

## 11. `item` / `iteration`

Dynamic Job でのみ利用可能。

### 11.1 `item`

現在の `foreach` 要素。

```yaml
foreach: ${{ needs.generate.outputs.candidates }}
key: ${{ item.id }}
with:
  candidate: ${{ item }}
```

### 11.2 `iteration`

入れ子 Dynamic Job で親 iteration context を参照するために使用する。

MVP では以下の考え方を採用する。

```text
iteration.current
iteration.parent
iteration.ancestors
```

- `iteration.current`: 現在の item と key
- `iteration.parent`: 直近親 iteration
- `iteration.ancestors`: 外側から内側までの ancestor 配列

ただし通常は `item` を使い、`iteration` は入れ子で親値が必要な場合にのみ使用する。

exact object schema は Dynamic Job 詳細設計で固定する。

## 12. `failure`

Retry 条件や failure handling でのみ利用可能。

```text
failure.category
failure.code
failure.message
failure.retryable
failure.details
```

例:

```yaml
retry:
  if: ${{ failure.retryable }}
```

failure が存在しない評価 context で `failure` を参照した場合は validation error。

## 13. `workflow` / `run` / `job`

実行 metadata の read-only context。

### 13.1 `workflow`

定義情報。

例:

```text
workflow.id
workflow.version
workflow.name
```

### 13.2 `run`

Workflow Run 情報。

例:

```text
run.id
run.priority
run.started_at
```

### 13.3 `job`

現在 Job の情報。

例:

```text
job.id
job.priority
job.attempt
```

MVP では metadata 全体を無制限公開せず、安定して利用できる field のみ公開する。

## 14. Job Input 構築

Job Input は Action 実行直前に 1 つの JSON-compatible object として確定する。

### 14.1 基本入力

`with` を Job Input の主定義とする。

```yaml
with:
  symbol: ${{ inputs.symbol }}
  candidate: ${{ item }}
  mode: research
```

### 14.2 object 全体の受け渡し

Job Input として既存 object 全体を渡すことも許可する。

概念記法:

```yaml
with:
  $base: ${{ needs.prepare.outputs.payload }}
  threshold: 0.8
```

`$base` は JobRunner reserved key とし、object でなければ error。

最終 Job Input は:

1. `$base` object を shallow copy
2. 同じ `with` 内の明示 field を上書き

とする。

deep merge は MVP では行わない。

理由:

- merge 規則を単純にする
- 意図しない nested overwrite を避ける

### 14.3 複数 base

MVP では `$base` は 1 個のみ。

複数 object merge が必要な場合は Action / CEL expression / JMESPath で明示的に構築する。

## 15. Job Input snapshot

Job が Runner に claim される前に、実行に必要な値を解決して canonical Job Input を作る。

含むもの:

- Workflow Input 由来値
- upstream Output 由来値
- Artifact reference
- env 値
- state current value
- Dynamic Job item
- literal 値

Secret value は前述のとおり snapshot の通常 payload には平文保存しない。

### 15.1 Retry

Retry では初回 Attempt と同一の canonical Job Input を使用する。

Retry 時点の新しい state / env / upstream Output で再評価して Input を変えない。

Input を変えたい場合は Retry ではなく新しい Job / Workflow Run とする。

## 16. Job Output

Action は JSON-compatible output を返す。

基本形:

```json
{
  "count": 120,
  "items": ["a", "b"]
}
```

### 16.1 Output schema

Job 定義に Output Schema がある場合、Action success 確定前に validation する。

Schema mismatch は Job failure。

Output Schema 未指定なら JSON-compatible であることのみ要求する。

### 16.2 JSON-compatible 判定

許可:

- null
- boolean
- integer / finite number
- string
- array
- object with string keys

拒否:

- Python object instance
- bytes
- datetime object の生値
- NaN / Infinity
- set / tuple の生値
- file handle

必要なら Action 側で string / object へ明示変換する。

### 16.3 Output サイズ

小さい JSON Output は Core が SQLite に保持する。

MVP では巨大 Output を自動的に file 化する機能を Core の必須責務にしない。

大きいデータは親システムへ保存し、Artifact reference を返す。

Output payload の最大許容サイズは system setting として設定可能にする。

既定値は実装詳細で固定するが、上限超過時は silent truncate せず failure とする。

## 17. Artifact 参照

Artifact は Output と独立した named reference として参照できる。

```yaml
with:
  source: ${{ needs.export.artifacts.dataset }}
```

式評価結果は Artifact 実体ではなく reference object。

概念形:

```json
{
  "artifact_id": "art_123",
  "name": "dataset",
  "uri": "parent://datasets/abc",
  "media_type": "application/x-parquet",
  "size": 123456
}
```

Action が Artifact 実体を読む方法は親システム側の責任。

## 18. null / missing

`null` と `missing` を区別する。

### 18.1 null

値が明示的に存在し、その値が null。

### 18.2 missing

field / key 自体が存在しない。

missing 値を必要な場所で参照した場合は原則 expression resolution error。

暗黙に null へ変換しない。

Optional Input など、missing が許可される field では schema 側で明示する。

## 19. 型変換

暗黙型変換は最小限にする。

禁止例:

- `"10"` を自動 integer 化
- `1` を自動 boolean 化
- object を自動 string 化

Workflow Input / Job Input Schema で要求型と一致しなければ validation error。

文字列埋め込み `${{ ... }}` のみ、scalar を string 化できる。

## 20. Expression Error

Expression evaluation error は構造化 failure として扱う。

代表 code:

```text
expression_compile_error
expression_context_error
expression_missing_value
expression_type_error
jmespath_compile_error
jmespath_evaluation_error
```

### 20.1 定義ロード時に検出できるもの

- CEL syntax
- JMESPath literal expression syntax
- 未知 top-level context
- 明らかな不正 field path
- 使用禁止 context

は Workflow load 時に reject。

### 20.2 実行時にしか分からないもの

- upstream Output に key がない
- state value が存在しない
- JMESPath 対象型が想定と異なる
- Secret missing

などは Job scheduling / Input resolution 時に failure とする。

## 21. 評価タイミング

式は field の意味に応じた時点で評価する。

| field | 評価時点 |
| --- | --- |
| Workflow concurrency group | Workflow Run 開始時 |
| Job `if` | Job が ready 候補になった時 |
| `foreach` | upstream dependency 解決後、展開時 |
| Dynamic `key` | expansion 時 |
| `order_by` | expansion 時 |
| Job `with` | Attempt 開始前 |
| Retry `if` | Attempt failure 確定後 |
| Reusable Workflow input | child Workflow Run 作成前 |

同一 Attempt の Job Input を Action 実行中に再評価しない。

## 22. `if` の結果

`if` は boolean を返さなければならない。

```yaml
if: ${{ needs.validate.outputs.count > 0 }}
```

string / integer / object を truthy / falsy として暗黙評価しない。

false の場合 Job は実行せず `completed / skipped` とする。

## 23. `success_if`

Action が正常終了して JSON Output を返した後、必要に応じて Output を使って Job の成功条件を判定できる。

例:

```yaml
success_if: ${{ outputs.failed_count < 3 }}
```

`success_if` context では `outputs` を追加で公開する。

- expression true: success
- expression false: failure
- expression error: validation / expression failure

`success_if` 未指定なら Action の正常終了 + Output validation success を Job success とする。

## 24. Expression helper

MVP で少なくとも以下を提供する。

```text
success()
failure()
cancelled()
always()
jmespath(value, expression)
```

### 24.1 状態 helper

`success()` / `failure()` / `cancelled()` / `always()` は Job condition 用 helper。

意味は GitHub Actions の考え方に可能な限り合わせるが、JobRunner の status / conclusion model を基準に実装する。

### 24.2 JSON helper

必要最小限として以下を許可してよい。

```text
to_json(value)
from_json(string)
```

ただし OSS CEL implementation で安全かつ型整合的に提供できる場合のみ採用する。

提供不能な場合、MVP 必須要件にはしない。

## 25. Canonicalization

Job Input / Output の比較・hash 用に JSON-compatible value を canonical JSON へ正規化する。

規則:

- UTF-8
- object key を安定順序で serialize
- unnecessary whitespace なし
- NaN / Infinity 禁止
- JSON scalar type を保持

この canonical representation は Job Input 同一性確認などに利用する。

用途を限定し、別の複雑な identity system は作らない。

## 26. Security

Expression Engine は以下を禁止する。

- filesystem access
- network access
- subprocess execution
- Python import
- arbitrary callable invocation
- reflection
- environment variable の無制限読取

CEL / JMESPath へ渡すのは JobRunner が明示的に構築した data context のみ。

## 27. Static validation

Workflow load 時に少なくとも以下を検証する。

1. `${{ ... }}` delimiter の整合
2. CEL compile
3. JMESPath literal compile
4. field ごとの許可 context
5. `needs` に存在しない Job の参照禁止
6. declared `needs` 外の upstream 参照禁止
7. Workflow Input 名の存在確認
8. Secret 名 syntax
9. `if` / Retry `if` / `success_if` が boolean expression として妥当か確認可能な範囲で検証
10. `$base` の使用位置
11. reserved key collision

Runtime start 時はさらに:

- caller Input type
- required Input
- Action Registry
- Runner Pool
- Secret Provider availability

を再検証する。

## 28. Reserved names

以下は JobRunner expression context / YAML 用の予約名とする。

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
$base
```

User-defined Input や Output に同名 field を含めること自体は許可するが、top-level context と衝突しない参照経路に置く。

例:

```text
inputs.job
needs.prepare.outputs.state
```

は許可。

## 29. 実装上の分離

Expression subsystem は以下を分離する。

```text
ExpressionCompiler
ExpressionEvaluator
ExpressionContextBuilder
JmespathAdapter
InputResolver
OutputValidator
CanonicalJson
```

Action / Scheduler / MCP Adapter が CEL library を直接呼ばない。

すべて JobRunner の Expression / Resolver layer を通す。

これにより CEL implementation 差し替え時の影響を局所化する。

## 30. 受入条件

最低限以下を自動テストする。

### Workflow Input

- required Input 成功 / 欠落 failure
- default 適用
- extra Input reject
- null と missing の区別
- type mismatch reject

### Expression

- scalar expression の型保持
- string interpolation
- syntax error reject
- unknown context reject
- declared `needs` 外参照 reject
- CEL boolean condition
- JMESPath filter
- JMESPath syntax error

### Job Input

- literal + expression の混在
- `$base` + field override
- `$base` non-object reject
- Retry で Input が変化しない
- state は Attempt 開始時 snapshot

### Output

- JSON-compatible success
- non-JSON value reject
- Output Schema mismatch failure
- NaN / Infinity reject
- size limit 超過時 failure

### Secrets

- Secret 正常解決
- missing Secret failure
- Secret が persisted Job Input / Event / log に平文出力されない

### Artifact

- named Artifact reference 解決
- 実体ではなく reference object が渡る

## 31. 後続詳細設計との境界

本書では以下を完全には定義しない。

- Dynamic Job の生成 ID / nested context exact schema
- Reusable Workflow output aggregation
- Artifact DB schema
- Workflow state DB transaction
- Service API request / response schema
- WebUI 表示方法

それぞれ対応する詳細設計で確定する。
