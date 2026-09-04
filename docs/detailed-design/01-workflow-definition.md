# 01. Workflow Definition 詳細設計

- Status: Draft v0.6
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML とMVP基盤依存の正規契約を定義する。式評価、Dynamic Job、Reusable Workflow、Retry/Recovery、Storage schema は各専用詳細設計に従う。

## 2. MVP Python / OSS依存

Pythonは **3.10以上**。

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

PyPI `cel-python` は `cloud-custodian/cel-python` を指す。

ライセンスは `ruamel.yaml/pydantic/jsonschema/jmespath = MIT`, `cel-python = Apache-2.0`。通常のCore依存として許容する。

`ruamel.yaml 0.19.0` は導入上の既知問題を避けるため対象外。

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
9. SQLite INTEGERへ保存する整数はsigned 64-bit範囲。

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

`env` はRun内immutable static JSON-compatible literal。Expression/Secret参照禁止。

## 5. Workflow ID / version / hash

Workflow IDは親登録名またはWorkflowResolver canonical reference。

`version`は `1..9223372036854775807` の整数。定義同一性はtyped definition canonical JSONのSHA-256 `definition_hash`でも確認する。

## 6. Workflow Input

標準型:

```text
string/integer/number/boolean/object/array
```

正規形:

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

- `type`: 必須
- `required`: default false
- `nullable`: default false
- `default`: 任意
- `schema`: optional JSON Schema

`null`許可は `nullable: true` だけを正規表現とする。

- caller null -> nullable=true必須
- `default:null` -> nullable=true必須
- required=trueはkey存在を要求。nullable=trueなら値null可
- optional+default無し=missing
- extra Input reject

Run開始時に検証しsnapshot。

## 7. Workflow `outputs`

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
```

Workflow success確定直前に評価。失敗は `workflow_output_invalid`。

Workflow Output全体はname mappingによるJSON object。PayloadStoreのinline/spillを使う。

## 8. Priority / Concurrency

Priorityはsigned 64-bit integer、default0、大きいほど高い。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

- group最終値: non-empty string必須
- trim/lowercase等の暗黙正規化無し
- case-sensitive完全一致
- max-runs: 1..signed64 max
- on-limit: queue|reject

Group不正ならRun row作成前にfailure。

## 9. Workflow `settings`

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
```

- max-dynamic-jobs: integer >=0, default1000
- external-lease-minutes: finite positive, default60
- external-on-lease-expiry: requeue|fail
- output-inline-threshold-bytes: positive integer, default4MiB

4MiBはOutput最大値ではなくinline/spill切替閾値。

## 10. Job ID / Dependencies

静的Job ID:

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

`[ ] /` はDynamic logical key用に予約。

`needs`はstringまたはarray。missing/self/duplicate/cycleはerror。Dynamic `foreach.parent`もDAG edge。

## 11. Action / Validator identity

Action Registry:

```text
action_id: non-empty string
action_version: non-empty string
```

Validator Registry:

```text
validator_id: non-empty string
validator_version: non-empty string
```

両Registryは親bootstrapでProcess起動時に再構築する。Version更新は親責任。integer等を暗黙string化しない。

Workflow Run開始時に参照するAction/ValidatorのID+versionをsnapshotする。

### Validator callable contract

Custom Validatorは親側のtrusted Python callable。

概念:

```python
def validate_result(value, input_data) -> ValidationResult:
    ...
```

`ValidationResult`:

```text
valid: boolean
code: non-empty string optional
message: string optional
retryable: boolean default false
details: JSON-compatible optional
```

規則:

- resultを変換・置換しない。read-only validationのみ
- Runtime Handle/Secret valueを渡さない
- 重い処理や長時間I/Oを行うValidationはnormal Jobとして実装する
- unhandled exceptionは `validator_exception`, retryable=false

MVP Validatorはtrusted lightweight callableとしてRuntime側で実行する。Sandbox/killable isolationは提供しない。

## 12. Job共通field

```yaml
jobs:
  example:
    name: Example Job
    needs: []
    runs-on: default
    executor: internal
    action: system.example
    validator: domain.validate_example
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

internalのみ。省略時System `default_runner_pool`、default文字列`default`。最終値non-empty string。未登録PoolはRun開始前error。

### `executor`

`internal|external_llm|human`。省略時internal。Reusableは`uses`で識別。

### `action`

internal必須。external/human/reusableは禁止。

### `validator`

Optional Validator Registry ID。

- internal: allowed
- external_llm: allowed
- human: forbidden
- reusable: forbidden（Child Workflow内で検証する）

未登録Validatorまたはversion解決不可はRun開始前error。

### `with`

最終Job Input object。Secret規則は`02/12`。

### `if`

boolean。default `${{ success() }}`。

### `success_if`

internal/externalのみ。Resultは任意JSON-compatible value。

### `continue-on-error`

boolean/CEL boolean、activation時snapshot。

### `priority`

signed 64-bit integer、default0。

### `timeout-minutes`

internalのみoptional、finite positive。external/human/reusableは禁止。

### `retry`

未指定automatic retry無し。詳細は`10`。

### Job `outputs`

`outputs.schema` はoptional JSON Schema。Action/External resultは任意JSON-compatible value。PayloadStoreでtransparent inline/spill。

### `external`

external_llmのみ。`lease-minutes` finite positive、`on-lease-expiry=requeue|fail`。

## 13. Result validation order

Internal/External resultの正規順序:

1. JSON-compatible / canonical JSON validation
2. optional `outputs.schema` JSON Schema
3. optional Custom Validator
4. optional `success_if`
5. SecretGuard
6. PayloadStore persistence
7. success terminal transition

Custom Validatorが `valid=false` の場合:

```text
category=validation
code=<validator code or domain_validation_failed>
message=<validator message>
retryable=<validator result>
details=<validator details>
```

## 14. Executor別constraint

### internal

`action` required。`validator/runs-on/timeout-minutes` optional。`uses/external` forbidden。

### external_llm

`validator/external` optional。`action/uses/runs-on/timeout-minutes` forbidden。

### human

`action/validator/uses/runs-on/success_if/external/timeout-minutes` forbidden。

### reusable

`uses` required literal。`action/validator/executor/runs-on/success_if/external/timeout-minutes` forbidden。

## 15. Dynamic syntax

Root `foreach: <expr>`、Nested `foreach.parent/items`。詳細は`05`。

## 16. Definition Snapshot

保存:

- workflow_id/version/name
- source YAML全文
- canonical typed JSON/hash
- Workflow Input snapshot
- Action ID/version snapshot
- Validator ID/version snapshot
- optional source_identity

## 17. 検証

Load:

- safe YAML/duplicate/merge/tag
- Input nullable/default
- Job/Dynamic cycle
- executor field constraint
- numeric finite/range
- CEL/JMESPath compile
- reusable syntax

Run start:

- Input
- Action/Validator ID+version
- Runner Pool
- concurrency group string
- reusable resolution/cycle
- Secret利用位置
- runtime settings

Failure時Run rowを作らない。

## 18. 受入条件

1. dependency install/import Python3.10
2. ruamel.yaml 0.19.1 lower bound
3. Input nullable
4. Action/Validator version non-empty string
5. Validator internal/external allowed, human/reusable reject
6. Validator false/exception/retryable result
7. validation order schema -> validator -> success_if -> SecretGuard
8. concurrency group/case
9. numeric finite/signed64
10. arbitrary JSON Output
11. transparent spill
12. internal timeout / others reject
13. Dynamic parent cycle
14. definition hash/action+validator snapshot
