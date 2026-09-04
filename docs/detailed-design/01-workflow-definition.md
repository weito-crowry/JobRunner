# 01. Workflow Definition 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML の正規契約を定義する。式評価、Dynamic Job、Reusable Workflow、Retry/Recovery、DB schema は各専用詳細設計に従う。

## 2. 基本原則

1. canonical authoring format は YAML。
2. safe loader を使用し、custom tag / 任意コード実行を禁止する。
3. mapping の重複 key と YAML merge key `<<` は error。
4. 未知 key は error。暗黙補正しない。
5. load 後は typed immutable `WorkflowDefinition` に正規化する。
6. Workflow Run 開始時に runtime dependency を含め再検証し、実使用定義を snapshot する。
7. 実行開始後の元 YAML 変更は既存 Workflow Run に反映しない。

## 3. トップレベル

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

必須は `name`, `version`, `jobs`。

- `inputs`: Workflow Input schema
- `env`: Run 内 immutable static values。**Secret参照も含め Secret 用途には使用しない**
- `outputs`: Workflow-level Output mapping
- `priority`: integer、既定 0、大きいほど高優先
- `concurrency`: Workflow Run concurrency
- `settings`: JobRunner 固有設定

## 4. Workflow ID / version / hash

Workflow ID は YAML の `name` ではなく、親システム登録名または WorkflowResolver の canonical reference から決める。

`version` は親側の明示 version。定義同一性は typed definition の canonical JSON を SHA-256 した `definition_hash` でも確認する。

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

標準型: `string / integer / number / boolean / object / array`。`null` 許可は schema で明示する。

Run開始時に required / extra / type / default を検証し、最終 Input を snapshot する。Run中は変更しない。

## 6. `env`

Workflow Run 内の固定値のみ。

```yaml
env:
  MODE: research
```

`${{ secrets.* }}` は `env` では禁止。Secret は `02-expression-and-inputs.md` の規則に従い internal Action Job の `with` でのみ参照する。

## 7. Workflow-level `outputs`

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
```

Workflow `success` 確定直前に `jobs` context で評価する。評価 failure は `workflow_output_invalid` とし Workflow conclusion を `failure` にする。

## 8. Workflow concurrency

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

- `group`: fixed string または式
- `max-runs`: integer >= 1
- `on-limit`: `queue | reject`、既定 `queue`

未指定時は無制限。

## 9. Workflow `settings`

MVP:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  max-job-output-bytes: 4194304
```

未知 key は error。優先順位は Job override > Workflow settings > System default。

## 10. Job ID

静的 Job ID:

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

`[` `]` `/` は Dynamic logical key 用に予約する。

## 11. Job 共通 field

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

### 11.1 `needs`

string または string array。存在しない Job、自己依存、duplicate、cycle は error。

Dynamic `foreach.parent` も依存 edge として DAG cycle 検証対象に含める。

### 11.2 `runs-on`

internal Job の Runner Pool。省略時は System `default_runner_pool`、既定文字列 `default`。最終解決 Pool が未登録なら Run開始前 error。

external/human/reusable では禁止。

### 11.3 `executor`

`internal | external_llm | human`。省略時 `internal`。Reusable Workflow Job は `uses` で識別し `executor` は書かない。

### 11.4 `action`

- internal: 必須
- external_llm / human / reusable: 禁止

### 11.5 `with`

Action / External / Human / Child Workflow Input。Secret参照の可否は `02` に従う。

### 11.6 `if`

boolean式。未指定は `${{ success() }}`。

### 11.7 `success_if`

internal / external_llm の Output validation 後に評価。human/reusable では禁止。

### 11.8 `continue-on-error`

boolean または CEL boolean。activation 時に snapshot。Job failure 自体は保持し、依存進行と Workflow conclusion で許容 failure として扱う。

### 11.9 `timeout-minutes`

positive number。未指定は timeout なし。

**MVPで execution timeout を適用するのは internal Job のみ。**

- external_llm: `timeout-minutes` 禁止。待機制御は External Lease が担当
- human: `timeout-minutes` 禁止。Review期限はMVPなし
- reusable: `timeout-minutes` 禁止。子 Workflow 各Jobのtimeoutに委ねる

### 11.10 `retry`

未指定は automatic retry なし。詳細は `10`。

### 11.11 Job `outputs`

internal/external result は JSON-compatible object。optional JSON Schema を指定可能。canonical UTF-8 JSON が `max-job-output-bytes` 超過なら `output_too_large`。

### 11.12 `external`

external_llm Jobのみ。

```yaml
external:
  lease-minutes: 120
  on-lease-expiry: fail
```

## 12. Executor別制約

### internal

- `action`: required
- `runs-on`: optional
- `timeout-minutes`: optional
- `uses/external`: forbidden

### external_llm

- `action/uses/runs-on/timeout-minutes`: forbidden
- `external`: optional

### human

- `action/uses/runs-on/success_if/external/timeout-minutes`: forbidden

### Reusable Workflow

```yaml
jobs:
  child:
    uses: ./workflows/child.yml
    with: {}
```

- `uses`: required, literal only
- `action/executor/runs-on/success_if/external/timeout-minutes`: forbidden

## 13. Dynamic Job syntax

Root:

```yaml
foreach: ${{ needs.generate.outputs.items }}
```

Nested:

```yaml
foreach:
  parent: evaluate
  items: ${{ iteration.parent.outputs.conditions }}
```

`parent` edge も DAG dependency として扱う。詳細は `05`。

## 14. Definition Snapshot

Run開始時に保存:

- workflow_id/version/name
- source YAML全文
- canonical typed definition JSON
- SHA-256 definition_hash
- Workflow Input snapshot
- Action ID/version snapshot
- optional source_identity

## 15. 検証段階

load時:

- safe YAML
- duplicate key / merge key / custom tag
- schema / unknown key
- Job ID / `needs` / Dynamic parent edge / cycle
- executor field constraint
- CEL/JMESPath compile
- reusable reference syntax
- retry/timeout/concurrency/settings

Run start時:

- Input
- Action ID/version
- resolved Runner Pool
- reusable resolution/cycle
- Secret利用位置
- runtime settings

失敗時は Workflow Run row を作らない。

## 16. 受入条件

1. duplicate/merge/custom tag拒否
2. `env` Secret参照拒否
3. Dynamic parent cycle検出
4. internal timeout許可
5. external/human/reusable timeout拒否
6. executor別field conflict
7. Workflow outputs
8. output size limit
9. definition hash deterministic
10. source変更後も既存Run snapshot不変
