# 01. Workflow Definition 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner の Workflow YAML の正規契約を定義する。式評価、Dynamic Job 展開、Reusable Workflow 実行、Retry/Recovery、DB schema は各専用詳細設計に従う。

## 2. 基本原則

1. canonical authoring format は YAML。
2. YAML は安全な loader で読み込み、任意 Python/Shell/custom tag を実行しない。
3. YAML mapping の重複 key は validation error。後勝ちで上書きしない。
4. YAML merge key `<<` は MVP では禁止し、定義を明示的に保つ。
5. 未知 key は schema validation error。黙って無視しない。
6. load 後は typed immutable `WorkflowDefinition` に正規化する。
7. Workflow Run 開始時に再検証し、その Run が使う定義を snapshot する。
8. 実行開始後の元 YAML 変更は既存 Workflow Run に反映しない。

## 3. トップレベル schema

```yaml
name: Example Workflow
version: 1
description: optional

inputs: {}
env: {}
outputs: {}
priority: 0
concurrency: {}
settings: {}
jobs: {}
```

| key | 必須 | 型 | 説明 |
| --- | --- | --- | --- |
| `name` | yes | string | 表示名 |
| `version` | yes | integer >= 1 | 親管理 version |
| `jobs` | yes | mapping | Job 定義 |
| `description` | no | string | 説明 |
| `inputs` | no | mapping | Workflow Input schema |
| `env` | no | mapping | Workflow Run 内の immutable static values |
| `outputs` | no | mapping | Workflow-level Output mapping。Reusable Workflow の返値にも使う |
| `priority` | no | integer | 既定 0。値が大きいほど高い |
| `concurrency` | no | mapping | Workflow Run concurrency |
| `settings` | no | mapping | JobRunner 固有設定 |

## 4. Workflow ID / version

Workflow ID は YAML の `name` ではなく、親システムの登録名または WorkflowResolver が解決した canonical reference から与える。同一 Runtime registry 内で一意とする。

`version` は人間・親システム向けの明示 version であり、定義同一性そのものには使わない。定義同一性は typed definition の canonical JSON を SHA-256 した `definition_hash` でも確認する。

## 5. Workflow Input

```yaml
inputs:
  symbol:
    type: string
    required: true
  timeframe:
    type: string
    default: M5
```

標準型は `string / integer / number / boolean / object / array`。`null` 許可は schema で明示する。

Input field:

- `type`: 必須
- `required`: boolean、既定 false
- `default`: 任意
- `description`: 任意
- `schema`: object/array 等の追加 JSON Schema

Workflow Run 開始時に required、extra field、型、default を検証し、最終 Input を snapshot する。Run 中は変更しない。

## 6. `env`

Workflow Run 内の immutable static values。

```yaml
env:
  MODE: research
```

Secret 値そのものは書かず `${{ secrets.NAME }}` を使用する。Secret の利用位置は `02-expression-and-inputs.md` に従う。

## 7. Workflow-level `outputs`

Workflow 完了時に公開する値を定義する。

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
  report: ${{ jobs.report.artifacts.report }}
```

- key は output 名。
- value は literal または `${{ ... }}`。
- 評価 context の `jobs` は `02-expression-and-inputs.md` の Workflow Output 用 context に従う。
- 通常 Workflow でも定義可能。Reusable Workflow 呼び出しでは親 Job Output として公開される。
- Workflow conclusion が `success` のとき評価する。評価失敗は `workflow_output_invalid` として Workflow conclusion を `failure` にする。

## 8. Workflow priority

```yaml
priority: 10
```

整数、既定 0。Workflow Run 開始時に snapshot し、実行中は Service API から変更可能。変更は実行中 Job を preempt しない。

## 9. Workflow concurrency

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

- `group`: 必須、固定 string または式
- `max-runs`: integer >= 1
- `on-limit`: `queue | reject`、既定 `queue`

未指定時は論理的 concurrency 制限なし。Runner Pool 数とは別概念。

## 10. Workflow `settings`

MVP で受理する key:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  max-job-output-bytes: 4194304
```

- `max-dynamic-jobs`: integer >= 0。既定 1000。
- `external-lease-minutes`: positive number。既定 60。
- `external-on-lease-expiry`: `requeue | fail`。既定 `requeue`。
- `max-job-output-bytes`: positive integer。既定 4 MiB = 4,194,304 bytes。canonical UTF-8 JSON の byte 長で判定する。

優先順位は Job override > Workflow settings > System default。未知 settings key は error。

## 11. Job ID

静的 Job ID は Workflow 内一意。

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

`[` `]` `/` は Dynamic Job の logical key に使用するため静的 Job ID では禁止する。

## 12. Job 共通 field

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
    success_if: ${{ outputs.ok == true }}
    continue-on-error: false
    priority: 0
    timeout-minutes: 30
    retry: {}
    outputs: {}
    foreach: null
    key: null
    order_by: null
    external: null
```

Reusable Workflow Job は `uses` を持つ。

### 12.1 `needs`

string 1件または string array。内部では順序付き tuple/list に正規化する。存在しない Job、自己依存、duplicate、DAG cycle は error。

Dynamic Job group や nested Dynamic Job の依存規則は `05-dynamic-jobs.md` に従う。

### 12.2 `runs-on`

`internal` Job の Runner Pool。

- optional。
- 省略時は System `default_runner_pool` を使う。System default は文字列 `default`。
- 最終的に解決された Pool が登録されていなければ Workflow Run 開始前 validation error。
- `external_llm` / `human` / Reusable Workflow Job では指定禁止。

`default` という Pool 自体が予約済みで必ず存在するわけではない。internal Job がそこへ解決される場合にのみ登録が必要。

### 12.3 `executor`

MVP:

- `internal`（省略時の既定）
- `external_llm`
- `human`

Reusable Workflow Job は `uses` で識別し、`executor` は指定しない。

### 12.4 `action`

- `internal`: 必須
- `external_llm`: 禁止
- `human`: 禁止
- Reusable Workflow (`uses`): 禁止

External/Human payload は `with` だけで構築する。

### 12.5 `with`

Action / executor / Child Workflow Input の定義。literal、array、object、expression を許可する。最終 Job Input は実行可能になった時点で解決・snapshotし、Retry では同じ永続 Input を再利用する。Secret の materialize は別扱い。

### 12.6 `if`

boolean 式。未指定時は `success()`。`if=false` の終端規則は `02` / `03` に従う。

### 12.7 `success_if`

Action または external result が正常に JSON Output を返した後に評価する optional boolean 式。

```yaml
success_if: ${{ outputs.failed_count < 3 }}
```

`internal` / `external_llm` で利用可能。`human` / Reusable Workflow Job では指定禁止。

### 12.8 `continue-on-error`

boolean または CEL expression、既定 false。Job activation 時に評価し boolean を snapshot する。Job failure を success に書き換えず、dependency / Workflow conclusion で「許容 failure」として扱う。

### 12.9 `priority`

integer、既定 0。値が大きいほど高い。

### 12.10 `timeout-minutes`

positive number。未指定時は timeout なし。External lease timeout とは別概念。

### 12.11 `retry`

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

未指定時 automatic retry なし。詳細は `10-retry-recovery-cancel.md`。

### 12.12 Job `outputs`

Action/external result の JSON object 全体が `needs.<job>.outputs`。optional `schema` で JSON Schema 検証できる。

```yaml
outputs:
  schema:
    type: object
    required: [count]
    properties:
      count: {type: integer}
```

canonical UTF-8 JSON が `max-job-output-bytes` を超える場合は `output_too_large` failure。大きいデータは Action/親側へ保存し Artifact として登録する。

### 12.13 `external`

`executor: external_llm` でのみ利用可能。

```yaml
external:
  lease-minutes: 120
  on-lease-expiry: fail
```

- `lease-minutes`: positive number
- `on-lease-expiry`: `requeue | fail`

未指定値は Workflow settings > System default を継承する。

## 13. Executor 別 field 制約

### internal

- `action`: required
- `uses`: forbidden
- `external`: forbidden
- `runs-on`: optional（default pool 解決）

### external_llm

- `action`: forbidden
- `uses`: forbidden
- `runs-on`: forbidden
- `external`: optional

### human

- `action`: forbidden
- `uses`: forbidden
- `runs-on`: forbidden
- `success_if`: forbidden
- `external`: forbidden

### Reusable Workflow

```yaml
jobs:
  child:
    uses: ./workflows/child.yml
    with: {}
```

- `uses`: required
- `action`, `executor`, `runs-on`, `success_if`, `external`: forbidden
- `uses` reference は literal のみ。詳細は `06-reusable-workflows.md`。

## 14. Dynamic Job の定義位置

Root Dynamic Job:

```yaml
jobs:
  evaluate:
    foreach: ${{ needs.generate.outputs.items }}
    key: ${{ item.id }}
    action: evaluate
```

Nested Dynamic Job は `foreach.parent/items` object form を使う。詳細は `05-dynamic-jobs.md`。

## 15. Definition Snapshot / hash

Workflow Run 開始時に少なくとも保存する。

- `workflow_id`, `version`, `name`
- source YAML 全文
- canonical typed definition JSON
- SHA-256 `definition_hash`
- Workflow Input snapshot
- 使用 Action ID/version snapshot
- optional `source_identity`

Hash は typed definition の Runtime 非依存項目を JSON key sort / UTF-8 / NaN・Infinity禁止で deterministic serialization して SHA-256。

## 16. Reload

元 Workflow YAML は親システムが reload 可能にしてよい。reload は新しい Workflow Run にのみ影響し、既存 Run の snapshot は不変。

## 17. 検証段階

### load 時

- safe YAML parse
- duplicate mapping key / merge key / custom tag rejection
- schema / unknown key
- Job ID / `needs` / cycle
- executor別 field constraint
- CEL/JMESPath compile
- Reusable Workflow reference syntax
- retry/timeout/concurrency/settings 型

### Workflow Run start 時

- Input validation
- Action ID/version存在
- resolved Runner Pool存在
- Reusable Workflow解決/cycle
- Secret reference policy
- runtime settings妥当性

検証 failure では Workflow Run row を作らない。

## 18. 受入条件

最低限以下をテストする。

1. valid YAML load
2. duplicate YAML key / merge key / custom tag rejection
3. unknown key rejection
4. Workflow Input required/default/type/extra
5. top-level Workflow `outputs`
6. internal `runs-on` explicit/default/unregistered
7. external/human/reusable field conflicts
8. `success_if` field
9. external lease override precedence
10. 4 MiB Job Output limit
11. action version snapshot
12. definition hash deterministic
13. source YAML change後も既存 Run snapshot不変
