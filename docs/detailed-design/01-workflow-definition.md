# 01. Workflow Definition 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML とMVP基盤依存の正規契約を定義する。式評価、Dynamic Job、Reusable Workflow、Retry/Recovery、Storage schema は各専用詳細設計に従う。

## 2. MVP Python / OSS依存

Pythonは **3.10以上**。

Coreの基本依存は以下へ固定する。

```text
ruamel.yaml >=0.19.1,<0.20   # YAML 1.2 / duplicate key検出
pydantic >=2.13,<3           # typed immutable models / request models
jsonschema >=4.26,<5         # optional Input/Output JSON Schema
cel-python >=0.5,<0.6        # CEL (PyPI cel-python / cloud-custodian/cel-python)
jmespath >=1.1,<2            # JSON projection/filter
```

MVP時点でPython 3.10で利用可能。ライセンスは `ruamel.yaml/pydantic/jsonschema/jmespath = MIT`, `cel-python = Apache-2.0` で、Coreへの通常依存として許容する。

`ruamel.yaml 0.19.0` は導入上の既知問題を避けるため対象外とし、0.19.1以上を使う。

YAMLは`ruamel.yaml`のduplicate key rejectを有効にし、merge key `<<` はJobRunner側でも明示rejectする。

Process/SQLite/JSON/UUID等はPython標準libraryを優先する。

## 3. YAML基本原則

1. canonical authoring formatはYAML。
2. safe loaderを使用しcustom tag/任意コード実行禁止。
3. mapping duplicate key、merge key `<<` はerror。
4. unknown keyはerror。
5. load後はtyped immutable `WorkflowDefinition`。
6. Run開始時にruntime dependency含め再検証し実使用定義をsnapshot。
7. 元YAML変更は既存Runへ反映しない。
8. 数値fieldはNaN/Infinityを拒否する。
9. SQLite INTEGERへ保存する整数はsigned 64-bit範囲に収める。

## 4. トップレベル

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

必須: `name/version/jobs`。

### `env`

Run内immutable static **literal** values。JSON-compatible literalのみで、`${{ ... }}` expression自体をMVPでは禁止する。Secret参照も禁止。

### `outputs`

Workflow-level Output mapping。literalまたはexpression。

## 5. Workflow ID / version / hash

Workflow IDは親登録名またはWorkflowResolver canonical reference。

`version`は `1..9223372036854775807` の整数。親管理versionであり、定義同一性はtyped definition canonical JSONのSHA-256 `definition_hash`も使用する。

## 6. Workflow Input

標準型:

```text
string/integer/number/boolean/object/array
```

Input fieldの正規形:

```yaml
inputs:
  name:
    type: string
    required: true
    nullable: false
    default: optional
    description: optional
    schema: optional
```

規則:

- `type`: 必須、上記標準型の1つ
- `required`: boolean、既定false
- `nullable`: boolean、既定false
- `default`: 任意
- `description`: string任意
- `schema`: JSON Schema object任意。object/array等の追加制約に使う

`null`を許可する正規方法は `nullable: true`。`type`を配列にする等の別表現はInput fieldでは許可しない。

- callerが`null`を渡す場合 `nullable=true` 必須
- `default: null` も `nullable=true` 必須
- `required=true` は「keyの存在」を要求し、`nullable=true`なら値nullは許可
- optional + default無しはmissing
- extra Inputはreject

Run開始時にrequired/extra/type/nullable/default/schemaを検証し、最終Inputをsnapshotする。

## 7. Workflow `outputs`

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
```

Workflow success確定直前に評価。失敗は `workflow_output_invalid`。

Workflow OutputもJob Outputと同じPayloadStore規則で、小さいJSONはSQLite、大きいJSONはfilesystemへ透過spillする。

## 8. Priority / Concurrency

Priorityはsigned 64-bit integer、既定0、大きいほど高い。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

Concurrency規則:

- `group`: literalまたはCEL。最終評価結果は**非空string必須**
- stringをtrim/lowercase等へ暗黙正規化しない
- group比較はUTF-8文字列の**case-sensitive完全一致**
- `max-runs`: `1..9223372036854775807` のinteger
- `on-limit`: `queue|reject`、既定`queue`

Group式がstring以外/null/emptyならRun start validation failureでRun rowを作らない。

## 9. Workflow `settings`

MVP:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
```

- `max-dynamic-jobs`: integer >=0、default1000、signed 64-bit範囲
- `external-lease-minutes`: finite positive number、default60分
- `external-on-lease-expiry`: `requeue|fail`
- `output-inline-threshold-bytes`: positive signed 64-bit integer、default4MiB

**4MiBは最大OutputサイズではなくSQLite inline保存からfilesystem blobへ切り替える閾値。** MVPにhiddenなJob Output最大サイズは設けない。

Unknown settingはerror。優先順位はWorkflow setting > System default。External専用値はJob `external` overrideが最優先。

## 10. Job ID / Dependencies

静的Job ID:

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

`[ ] /` はDynamic logical key用に予約。

`needs`はstringまたはarray。存在しないJob、self、duplicate、cycleはerror。

Dynamic `foreach.parent`もDAG dependency edgeとしてcycle検証対象。

## 11. Action identity

Action Registryの正規identity:

```text
action_id: non-empty string
action_version: non-empty string
```

Action versionは親システムが更新責任を持つ。integer等を暗黙string変換せず、Registry登録APIでstringを要求する。

Workflow Run開始時に参照Actionの `action_id + action_version` をsnapshotする。

## 12. Job共通field

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

### `runs-on`

internalのみ。省略時System `default_runner_pool`、default文字列`default`。最終値は非空string。未登録PoolはRun開始前error。

### `executor`

`internal|external_llm|human`。省略時internal。Reusableは`uses`で識別。

### `action`

internal必須。external/human/reusableは禁止。値はRegistryのnon-empty `action_id` string。

### `with`

最終Job Input objectを構築。Secret規則は`02/12`。

### `if`

boolean。未指定 `${{ success() }}`。

### `success_if`

internal/externalのみ。**Action/External resultは任意のJSON-compatible valueでよい**ため、`outputs` context自体がscalar/list/object/nullのいずれにもなり得る。

### `continue-on-error`

boolean/CEL boolean、activation時snapshot。

### `priority`

signed 64-bit integer、既定0。

### `timeout-minutes`

internalのみoptional。finite positive number。external/human/reusableは禁止。

### `retry`

未指定automatic retry無し。数値規則は`10`。

### Job `outputs`

`outputs` fieldはoptional JSON Schema定義用。

```yaml
outputs:
  schema:
    type: array
```

実際のAction/External resultは **JSON-compatible value全般**を許可し、`needs.<job>.outputs` はそのvalueをそのまま返す。

保存はPayloadStoreによりtransparent inline/spill。大きいJSONを理由にJob failureへしない。

### `external`

external_llmのみ:

```yaml
external:
  lease-minutes: 120
  on-lease-expiry: fail
```

`lease-minutes`はfinite positive number。

## 13. Executor別constraint

### internal

`action` required。`runs-on/timeout-minutes` optional。`uses/external` forbidden。

### external_llm

`action/uses/runs-on/timeout-minutes` forbidden。`external` optional。

### human

`action/uses/runs-on/success_if/external/timeout-minutes` forbidden。

### reusable

`uses` required literal。`action/executor/runs-on/success_if/external/timeout-minutes` forbidden。

## 14. Dynamic syntax

Root:

```yaml
foreach: ${{ needs.generate.outputs }}
```

Nested:

```yaml
foreach:
  parent: evaluate
  items: ${{ iteration.parent.outputs }}
```

詳細は`05`。

## 15. Definition Snapshot

保存:

- workflow_id/version/name
- source YAML全文
- canonical typed JSON/hash
- Workflow Input snapshot
- Action ID/version snapshot
- optional source_identity

## 16. 検証

load:

- safe YAML / duplicate / merge / custom tag
- schema/unknown key
- Input nullable/default schema
- Job/Dynamic dependency cycle
- executor field constraint
- numeric finite/range
- CEL/JMESPath compile
- reusable syntax
- retry/internal timeout/settings

Run start:

- Input
- Action ID/version string
- Runner Pool
- concurrency group result string
- reusable resolution/cycle
- Secret利用位置
- runtime settings

失敗時Run rowを作らない。

## 17. 受入条件

1. dependency versions/import on Python3.10
2. ruamel.yaml 0.19.1 lower bound
3. duplicate/merge/tag rejection
4. Input nullable true/false/default null
5. env literal-only/expression reject
6. Action version non-empty string
7. concurrency group non-empty string/case-sensitive
8. numeric finite/signed64 boundary
9. arbitrary JSON Output scalar/list/object/null
10. inline threshold 4MiBとtransparent spill
11. Outputにhidden max無し
12. internal timeout / others reject
13. Dynamic parent cycle
14. definition hash deterministic
