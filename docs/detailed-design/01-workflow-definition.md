# 01. Workflow Definition 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner の Workflow YAML を、実装時に解釈差が生じない粒度まで定義する。

対象は以下。

- Workflow YAML の基本構造
- Workflow / Job の識別
- Workflow Input
- Job 定義
- `needs`
- `runs-on`
- `action`
- `executor`
- `with`
- `if`
- `continue-on-error`
- Retry / timeout の定義位置
- Dynamic Job / Reusable Workflow の定義位置
- Workflow concurrency の定義位置
- Workflow Run 開始時の Definition Snapshot
- 静的検証
- 未知キー、型不一致、参照不正時の扱い

Expression の評価規則そのもの、Dynamic Job の展開アルゴリズム、Reusable Workflow の実行詳細は各専用詳細設計で定義する。

## 2. 基本原則

1. Workflow の canonical authoring format は YAML とする。
2. YAML は任意コード実行用の DSL にしない。
3. 実処理は Action Registry に登録された Action を参照する。
4. Workflow YAML はロード時に厳格検証する。
5. Workflow Run 開始時にも、現在の Action Registry / Runner Pool / Input を含めて再検証する。
6. 実行開始後は元 YAML の変更を反映しない。
7. Workflow Run は開始時点の Workflow 定義 Snapshot を保持する。
8. 未知キーは原則エラーとし、黙って無視しない。
9. 型不一致や参照不正を暗黙変換や推測で補正しない。
10. GitHub Actions と同義の概念がある場合は、可能な限り同じ名称を優先する。

## 3. Workflow YAML のトップレベル

MVP のトップレベルキーは以下を基本とする。

```yaml
name: Example Workflow
version: 1

inputs: {}
env: {}

priority: 0

concurrency: {}

settings: {}

jobs: {}
```

### 3.1 必須キー

| キー | 必須 | 型 | 説明 |
| --- | --- | --- | --- |
| `name` | 必須 | string | Workflow 表示名 |
| `version` | 必須 | integer >= 1 | Workflow 定義の親管理 version |
| `jobs` | 必須 | mapping | Job 定義集合 |

### 3.2 任意キー

| キー | 型 | 説明 |
| --- | --- | --- |
| `inputs` | mapping | Workflow Input schema |
| `env` | mapping | Workflow Run 内の immutable static values |
| `priority` | integer | Workflow Run の既定 priority |
| `concurrency` | mapping | Workflow Run concurrency 制御 |
| `settings` | mapping | JobRunner 固有の Workflow 単位設定 |
| `description` | string | 説明文 |

未知トップレベルキーは schema validation error とする。

## 4. Workflow ID

Workflow ID はファイルパスまたは親システム登録名から Runtime が与える。

YAML 内の `name` は表示名であり、Workflow ID そのものとして扱わない。

例:

```text
workflow_id = fiction.style-analysis
name        = Fiction Style Analysis
version     = 3
```

親システムは同一 Runtime 内で Workflow ID を一意に登録しなければならない。

## 5. Workflow version

`version` は正の整数とする。

```yaml
version: 3
```

用途:

- 人間向けの明示的な Workflow 定義 version
- Workflow Run 履歴表示
- 親システムによる互換性管理

JobRunner は `version` だけで Workflow 定義の同一性を判断しない。

Workflow Run では YAML 全体から計算した Definition Hash も保存し、実際に同一の定義かを比較できるようにする。

## 6. Workflow Input

### 6.1 基本形式

```yaml
inputs:
  symbol:
    type: string
    required: true

  timeframe:
    type: string
    required: false
    default: M5

  max_candidates:
    type: integer
    required: false
    default: 100
```

### 6.2 MVP の標準型

少なくとも以下を扱う。

- `string`
- `integer`
- `number`
- `boolean`
- `object`
- `array`

`null` 許可は schema 上で明示する。

### 6.3 Input field

各 Input は少なくとも以下を持てる。

| キー | 型 | 説明 |
| --- | --- | --- |
| `type` | string | 必須 |
| `required` | boolean | 既定 `false` |
| `default` | any | 任意 |
| `description` | string | 任意 |
| `schema` | object | `object` / `array` 等の追加 JSON Schema |

### 6.4 検証

Workflow Run 開始時に Input 全体を検証する。

- required 不足: start failure
- 未定義 Input: start failure
- 型不一致: start failure
- default の型不一致: definition validation failure

最終 Input は Workflow Run 開始時に Snapshot として固定する。

Workflow Run 開始後に変更しない。

## 7. `env`

Workflow Run 内で共有する immutable static values とする。

```yaml
env:
  MODE: research
  REGION: jp
```

用途:

- Workflow 全体の固定値
- 各 Job からの参照

`env` は mutable Workflow State とは別物とする。

Workflow Run 中の変更は禁止する。

Secret value を `env` に直接書くことは推奨しない。Secret は `${{ secrets.NAME }}` 参照を利用する。

## 8. Workflow priority

```yaml
priority: 10
```

- integer
- 値が大きいほど高 priority とする
- 未指定時は `0`
- Workflow Run 開始時の既定 priority として Snapshot する
- 実行中の Workflow Run priority は Runtime API から変更可能
- priority 変更は現在実行中 Job を preempt しない

## 9. Workflow concurrency

トップレベルに定義する。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

### 9.1 field

| キー | 必須 | 説明 |
| --- | --- | --- |
| `group` | 必須 | CEL expression または固定 string |
| `max-runs` | 必須 | integer >= 1 |
| `on-limit` | 任意 | `queue` / `reject`。既定 `queue` |

`concurrency` 未指定時は論理的な Workflow Run concurrency 制限なし。

Runner Pool の物理 Runner 数とは別概念とする。

## 10. Workflow `settings`

GitHub Actions に直接対応しない JobRunner 固有設定をまとめる領域とする。

MVP で想定する代表例:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
```

システム既定値より Workflow 単位設定を優先する。

詳細な優先順位は各機能の詳細設計で定義する。

未知 `settings` key は原則 validation error とし、実装済み設定だけを受理する。

## 11. `jobs`

`jobs` は Job ID を key とする mapping。

```yaml
jobs:
  generate:
    action: fx.generate_candidates

  evaluate:
    needs: [generate]
    action: fx.evaluate_candidate
```

### 11.1 Job ID

Job ID は Workflow 定義内で一意でなければならない。

MVP の Job ID 制約:

- 1 文字以上
- ASCII letter / digit / `_` / `-` を許可
- 先頭は letter または `_`
- 大文字小文字を区別する
- Dynamic Job の内部展開記法に使う `[` `]` は静的 Job ID に使用不可

推奨正規表現:

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

## 12. Job 共通 field

MVP の Job 共通 field は以下を基本とする。

```yaml
jobs:
  example:
    name: Example Job
    needs: []
    runs-on: default
    executor: internal
    action: system.example
    with: {}
    if: ${{ success() }}
    continue-on-error: false
    priority: 0
    timeout-minutes: 30
    retry: {}
    outputs: {}
    foreach: null
    key: null
    order_by: null
```

### 12.1 `name`

表示名。任意。

未指定時は Job ID を表示名として利用できる。

### 12.2 `needs`

上流依存 Job を指定する。

```yaml
needs: [generate, validate]
```

string 1件または string array を受理してよいが、内部モデルでは tuple/list に正規化する。

未指定時は依存なし。

静的検証:

- 存在しない Job ID: error
- 自己依存: error
- DAG cycle: error
- duplicate entry: error

Dynamic Job template 名への依存は、その template から生成された Job 群全体への依存として解釈する。詳細は Dynamic Job 詳細設計で定義する。

### 12.3 `runs-on`

internal executor が利用する Runner Pool 名。

```yaml
runs-on: heavy
```

- string
- internal Job では必須を基本とする
- 親システム登録済み Runner Pool と一致しなければ Workflow Run 開始前に error

`external_llm` / `human` executor では不要。

### 12.4 `executor`

MVP:

- `internal`
- `external_llm`
- `human`

未指定時は `internal`。

### 12.5 `action`

親システム Action Registry の Action ID。

```yaml
action: fx.run_backtest
```

- `internal`: 必須
- `external_llm`: Job payload を準備する Action を使う設計を許容するが、必須/任意の exact rule は External Executor 詳細設計で確定する
- `human`: 原則不要

Workflow Run 開始時に Action Registry の存在と version を確認する。

### 12.6 `with`

Action / executor へ渡す Job Input 定義。

```yaml
with:
  symbol: ${{ inputs.symbol }}
  candidate: ${{ needs.generate.outputs.candidate }}
  mode: strict
```

値は以下を許可する。

- scalar
- array
- object
- `${{ ... }}` expression

最終 Job Input は Job が実行可能になった時点で評価し、Attempt 開始前に Snapshot する。

Retry では同じ Job Input Snapshot を使用する。

### 12.7 `if`

Job を実行する条件。

```yaml
if: ${{ needs.validate.outputs.ok == true }}
```

未指定時は通常の依存成功条件を適用する。

条件評価結果が false の場合、Job は `completed / skipped` に確定する。

Expression の context / helper は Expression 詳細設計で定義する。

### 12.8 `continue-on-error`

```yaml
continue-on-error: true
```

- boolean または CEL expression
- 既定 `false`

Job 自身の conclusion が failure である事実は保持する。

`continue-on-error: true` は失敗を成功へ書き換える機能ではなく、Workflow 全体や依存進行で失敗を許容するための制御とする。

### 12.9 `priority`

```yaml
priority: 20
```

- integer
- 未指定 `0`
- 値が大きいほど高 priority

Runner が Job を選択する際は、上位仕様で定義した Workflow Run priority の次に Job priority を評価する。

### 12.10 `timeout-minutes`

```yaml
timeout-minutes: 120
```

- positive number
- 未指定時は timeout なし
- hidden default timeout は設けない

External lease timeout とは別概念。

### 12.11 `retry`

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
```

未指定時は automatic retry なし。

詳細な Retry 判定順、backoff、manual retry は Retry / Recovery 詳細設計で定義する。

### 12.12 `outputs`

Job Output の公開名や schema を定義する領域。

MVP は Action が返す JSON object をそのまま `needs.<job>.outputs` から参照可能とする。

必要に応じて output schema を指定できる。

例:

```yaml
outputs:
  schema:
    type: object
    required: [count]
    properties:
      count:
        type: integer
```

Output mapping の exact syntax は Inputs / Expression 詳細設計で確定する。

## 13. Dynamic Job 定義位置

Dynamic Job は通常 Job 定義へ `foreach` / `key` / `order_by` を追加する。

```yaml
jobs:
  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    order_by:
      - expr: ${{ item.priority }}
        direction: desc
    action: fx.evaluate_candidate
    with:
      candidate: ${{ item }}
```

Workflow Definition parser はこれらの field を保持するが、展開処理自体は Runtime の Dynamic Job subsystem が担当する。

動的 Job の入れ子段数には固定上限を設けない。

既定の Workflow Run 全体 Dynamic Job 生成数上限は 1000。

## 14. Reusable Workflow 定義位置

Reusable Workflow は Job の一種として扱う。

Action と Child Workflow 呼び出しを同時指定しない。

仮の canonical syntax:

```yaml
jobs:
  child:
    uses: workflows/common-validation.yaml
    with:
      source: ${{ inputs.source }}
```

`uses` がある Job は Reusable Workflow Job と解釈する。

MVP では以下を禁止する。

- `action` と `uses` の同時指定
- `executor` と `uses` の矛盾指定
- Child Workflow circular reference

`uses` の reference 形式、version 解決、Input / Output mapping は Reusable Workflow 詳細設計で確定する。

## 15. Job executor 別の field 制約

### 15.1 internal

原則:

```yaml
executor: internal
runs-on: default
action: parent.action
```

- `action`: 必須
- `runs-on`: 必須

### 15.2 external_llm

原則:

```yaml
executor: external_llm
```

- `runs-on`: 禁止
- Lease / claim / submit は Runtime が管理
- External task payload の作り方は専用詳細設計で定義

### 15.3 human

原則:

```yaml
executor: human
```

- `runs-on`: 禁止
- `action`: 原則禁止
- outcome は MVP では approve / reject

### 15.4 reusable workflow

```yaml
uses: workflows/child.yaml
```

- `action`: 禁止
- `runs-on`: 禁止
- Child Workflow Run が独自に Runner を利用する

## 16. Definition load

Workflow YAML load 処理は以下の順序とする。

1. UTF-8 text 読込
2. YAML parse
3. top-level shape validation
4. typed schema validation
5. Job ID validation
6. Job field combination validation
7. static `needs` resolution
8. DAG cycle detection
9. Expression compile validation
10. JMESPath compile validation
11. Reusable Workflow reference validation可能範囲
12. immutable `WorkflowDefinition` 構築
13. canonical representation 作成
14. Definition Hash 計算

## 17. Typed immutable model

ロード後の Workflow は mutable dict を直接 Runtime へ渡さない。

概念型:

```python
@dataclass(frozen=True)
class WorkflowDefinition:
    workflow_id: str
    name: str
    version: int
    inputs: Mapping[str, InputDefinition]
    env: Mapping[str, JsonValue]
    priority: int
    concurrency: ConcurrencyDefinition | None
    settings: WorkflowSettings
    jobs: Mapping[str, JobDefinition]
    source_yaml: str
    definition_hash: str
```

実装では Pydantic frozen model 等を利用してよい。

具体的ライブラリ選定は実装時に最新互換性を確認する。

## 18. Canonical representation / Definition Hash

Definition Hash は Workflow YAML の意味的内容比較に使う。

目的:

- Workflow Run がどの定義で開始されたか識別
- 同一 Workflow Run 内の再利用条件の一部
- 監査・デバッグ

Hash 計算時は YAML のコメントやインデント差だけで値が変わらないよう、typed model から canonical JSON を生成して hash する。

基本:

1. typed `WorkflowDefinition` から Runtime 非依存の定義項目だけ抽出
2. JSON key sort
3. UTF-8
4. deterministic serialization
5. SHA-256

`source_yaml` 自体も Snapshot として別途保存する。

つまり以下の両方を保持する。

- 人間が読める実際の YAML Snapshot
- 比較用 Definition Hash

## 19. Workflow Run start validation

Definition load 時に通っていても、Workflow Run 開始時に再検証する。

理由:

- Action Registry が変化している可能性
- Runner Pool 登録が変化している可能性
- Reusable Workflow 登録が変化している可能性
- Runtime Input がこの時点で初めて与えられる

開始時検証:

1. Workflow Input
2. Action ID 存在
3. Action version 解決
4. `runs-on` Pool 存在
5. Reusable Workflow 存在
6. Reusable Workflow cycle
7. runtime expression の開始時評価可能部分
8. concurrency group compile / initial evaluation
9. Workflow settings の system limit 整合

1件でも failure なら Workflow Run record を実行中として開始しない。

開始失敗を履歴として保存するかどうかは Persistence / Service API 詳細設計で確定する。

## 20. Workflow Run Definition Snapshot

Workflow Run 作成時に少なくとも以下を保存する。

```text
workflow_id
workflow_name
workflow_version
source_yaml
definition_hash
input_snapshot
env_snapshot
resolved_action_versions
source_identity (optional)
created_at
```

### 20.1 Action version snapshot

静的に参照される Action は Workflow Run 開始時に version を解決して保存する。

Dynamic Job でも元 template の Action ID / version を基準にする。

Reusable Workflow 内 Action は Child Workflow Run 側で別途 Snapshot する。

## 21. Retry / Resume と Definition

Retry / Resume では元 Workflow Run の Definition Snapshot を使う。

元 Workflow YAML ファイルを再読込して置き換えない。

Action version は現在 Registry と照合する。

元 Workflow Run が要求する Action version が現在 Registry に存在しない場合は fail-closed とする。

## 22. YAML reload

親システム稼働中に Workflow YAML の再読込を可能にする。

- reload 成功: 新しい Workflow Run から新定義を利用
- reload failure: 現在有効な最後の正常定義を維持
- 実行中 Workflow Run: 影響なし

Python Action code の reload は親システムの開発運用に委ねる。

## 23. Unknown / deprecated field

### 23.1 Unknown field

原則 error。

理由:

```yaml
continute-on-error: true
```

のような typo を黙って無視すると危険なため。

### 23.2 Deprecated field

将来 field rename が必要になった場合は、明示的な deprecated alias と warning を期間限定で提供してよい。

暗黙 alias は作らない。

## 24. Static validation error

Validation error は path を必須で持つ。

例:

```json
{
  "code": "unknown_runner_pool",
  "path": "jobs.evaluate.runs-on",
  "message": "Runner Pool 'gpu' is not registered"
}
```

最低限:

```text
code
path
message
details
```

複数 error を一度に列挙可能にする。

1件目だけで停止する必要はないが、依存する後続検証で意味のない cascade error を大量生成しない。

## 25. 完全な最小例

```yaml
name: Strategy Evaluation
version: 1

description: Candidate generation and evaluation

inputs:
  symbol:
    type: string
    required: true
  limit:
    type: integer
    default: 100

env:
  MODE: research

priority: 0

concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue

settings:
  max-dynamic-jobs: 1000

jobs:
  generate:
    name: Generate candidates
    runs-on: default
    action: fx.generate_candidates
    with:
      symbol: ${{ inputs.symbol }}
      limit: ${{ inputs.limit }}

  evaluate:
    name: Evaluate candidate
    needs: [generate]
    runs-on: heavy
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    order_by:
      - expr: ${{ item.priority }}
        direction: desc
      - expr: ${{ item.id }}
        direction: asc
    action: fx.evaluate_candidate
    with:
      candidate: ${{ item }}
    retry:
      max-attempts: 3
      if: ${{ failure.retryable }}
      backoff:
        initial-seconds: 5
        max-seconds: 60

  aggregate:
    name: Aggregate results
    needs: [evaluate]
    runs-on: default
    if: ${{ always() }}
    action: fx.aggregate
    with:
      results: ${{ needs.evaluate.outputs }}
```

## 26. 本書で未確定とせず、後続詳細設計へ委譲する項目

以下は未決定という意味ではなく、責務分離のため別文書で exact contract を定義する。

- CEL / JMESPath context と評価順
- `needs.*` / `item` / `failure` の exact object shape
- Dynamic Job 展開時の exact ID / nested context
- Reusable Workflow `uses` reference の exact syntax
- Retry / Recovery state transition
- Persistence schema
- Service API request / response
- External LLM task payload
- Human Review payload

## 27. 実装時の受入条件

最低限、以下をテストする。

1. 正常 YAML を immutable `WorkflowDefinition` へ変換できる
2. unknown key を拒否する
3. duplicate / invalid Job ID を拒否する
4. missing `needs` target を拒否する
5. DAG cycle を拒否する
6. 未登録 Action を Workflow Run start 時に拒否する
7. 未登録 Runner Pool を Workflow Run start 時に拒否する
8. Input required / type / default を検証する
9. invalid CEL / JMESPath を拒否する
10. invalid executor / field combination を拒否する
11. YAML の空白・コメント差で Definition Hash が変わらない
12. 意味的な定義変更で Definition Hash が変わる
13. Workflow Run 開始後に元 YAML を変更しても Run Snapshot が変わらない
14. Retry / Resume が元 Definition Snapshot を使用する
15. Action version mismatch を fail-closed で拒否する
