# 01. Workflow Definition 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML とMVP基盤依存の正規契約を定義する。式評価、Dynamic Job、Reusable Workflow、Retry/Recovery、Storage schema は各専用詳細設計に従う。

## 2. MVP Python / OSS依存

Pythonは **3.10以上**。

Coreの基本依存は以下へ固定する。

```text
ruamel.yaml >=0.19,<0.20     # YAML 1.2 / duplicate key検出
pydantic >=2.13,<3           # typed immutable models / request models
jsonschema >=4.26,<5         # optional Input/Output JSON Schema
cel-python >=0.5,<0.6        # CEL
jmespath >=1.1,<2            # JSON projection/filter
```

MVP時点でいずれもPython 3.10で利用可能。YAMLは`ruamel.yaml`のduplicate key rejectを有効にし、merge key `<<` はJobRunner側でも明示rejectする。

Process/SQLite/JSON/UUID等はPython標準libraryを優先する。

## 3. YAML基本原則

1. canonical authoring formatはYAML。
2. safe loaderを使用しcustom tag/任意コード実行禁止。
3. mapping duplicate key、merge key `<<` はerror。
4. unknown keyはerror。
5. load後はtyped immutable `WorkflowDefinition`。
6. Run開始時にruntime dependency含め再検証し実使用定義をsnapshot。
7. 元YAML変更は既存Runへ反映しない。

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

`version`は親管理version。定義同一性はtyped definition canonical JSONのSHA-256 `definition_hash`も使用。

## 6. Workflow Input

標準型:

```text
string/integer/number/boolean/object/array
```

`null`許可は明示。Run開始時にrequired/extra/type/defaultを検証しsnapshot。

## 7. Workflow `outputs`

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
```

Workflow success確定直前に評価。失敗は `workflow_output_invalid`。

Workflow OutputもJob Outputと同じPayloadStore規則で、小さいJSONはSQLite、大きいJSONはfilesystemへ透過spillする。

## 8. Priority / Concurrency

Priorityはinteger、既定0、大きいほど高い。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

`on-limit = queue|reject`。

## 9. Workflow `settings`

MVP:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
```

- `max-dynamic-jobs`: integer >=0、default1000
- external lease: default60分 / `requeue`
- `output-inline-threshold-bytes`: positive integer、default4MiB

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

## 11. Job共通field

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

internalのみ。省略時System `default_runner_pool`、default文字列`default`。未登録PoolはRun開始前error。

### `executor`

`internal|external_llm|human`。省略時internal。Reusableは`uses`で識別。

### `action`

internal必須。external/human/reusableは禁止。

### `with`

最終Job Input objectを構築。Secret規則は`02/12`。

### `if`

boolean。未指定 `${{ success() }}`。

### `success_if`

internal/externalのみ。**Action/External resultは任意のJSON-compatible valueでよい**ため、`outputs` context自体がscalar/list/object/nullのいずれにもなり得る。

### `continue-on-error`

boolean/CEL boolean、activation時snapshot。

### `timeout-minutes`

internalのみoptional。external/human/reusableは禁止。

### `retry`

未指定automatic retry無し。

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

## 12. Executor別constraint

### internal

`action` required。`runs-on/timeout-minutes` optional。`uses/external` forbidden。

### external_llm

`action/uses/runs-on/timeout-minutes` forbidden。`external` optional。

### human

`action/uses/runs-on/success_if/external/timeout-minutes` forbidden。

### reusable

`uses` required literal。`action/executor/runs-on/success_if/external/timeout-minutes` forbidden。

## 13. Dynamic syntax

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

## 14. Definition Snapshot

保存:

- workflow_id/version/name
- source YAML全文
- canonical typed JSON/hash
- Workflow Input snapshot
- Action ID/version snapshot
- optional source_identity

## 15. 検証

load:

- safe YAML / duplicate / merge / custom tag
- schema/unknown key
- Job/Dynamic dependency cycle
- executor field constraint
- CEL/JMESPath compile
- reusable syntax
- retry/internal timeout/settings

Run start:

- Input
- Action ID/version
- Runner Pool
- reusable resolution/cycle
- Secret利用位置
- runtime settings

失敗時Run rowを作らない。

## 16. 受入条件

1. dependency versions/import on Python3.10
2. duplicate/merge/tag rejection
3. env literal-only/expression reject
4. arbitrary JSON Output scalar/list/object/null
5. inline threshold 4MiBとtransparent spill
6. Outputにhidden max無し
7. internal timeout / others reject
8. Dynamic parent cycle
9. definition hash deterministic
