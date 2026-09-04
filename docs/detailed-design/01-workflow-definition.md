# 01. Workflow Definition 詳細設計

- Status: Draft v0.7
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML、型、JSON Schema、canonical serialization、Action/Validator定義の正規契約を定義する。

## 2. MVP Python / OSS依存

Python **>=3.10**。

```text
ruamel.yaml >=0.19.1,<0.20
pydantic >=2.13,<3
jsonschema >=4.26,<5
cel-python >=0.5,<0.6
jmespath >=1.1,<2
```

PyPI `cel-python` は cloud-custodian/cel-python。

ライセンス: `ruamel.yaml/pydantic/jsonschema/jmespath=MIT`, `cel-python=Apache-2.0`。

`ruamel.yaml 0.19.0` は導入上の既知問題を避け対象外。

Process/SQLite/JSON/UUID/hashlibはPython標準libraryを優先。

## 3. YAML基本原則

1. Canonical authoring format=YAML 1.2。
2. `ruamel.yaml` safe loaderをYAML 1.2 modeで使用。
3. custom tag/任意object construction禁止。
4. duplicate mapping key reject。
5. merge key `<<` reject。
6. unknown key reject。
7. load後typed immutable `WorkflowDefinition`。
8. 数値NaN/Infinity reject。
9. SQLite INTEGER対象はsigned64範囲。
10. Run開始時再検証し実使用Definition snapshot固定。

YAML 1.1の暗黙boolean等へfallbackしない。

## 4. Canonical JSON v1

Definition hash、Input digest、Output digest、reuse key等で共通利用するserializationを `jobrunner.canonical-json.v1` と呼ぶ。

Python標準`json.dumps`相当で以下を固定:

```python
json.dumps(
    value,
    ensure_ascii=False,
    sort_keys=True,
    separators=(",", ":"),
    allow_nan=False,
)
```

その結果のUnicode stringをUTF-8 encodeする。

追加規則:

- JSON object keyはstringのみ
- duplicate keyは構築前段で禁止
- NaN/Infinity禁止
- Python独自型(datetime, Decimal, bytes, tuple等)を暗黙変換しない
- tuple等内部typed modelはJSON treeへ明示変換してからcanonicalize
- Unicode normalization(NFC等)を暗黙適用しない
- `1` と `1.0` は異なるJSON表現として扱う

Hash/digest:

```text
SHA-256(canonical-json-v1 UTF-8 bytes)
lowercase hex 64 chars
```

Definition/reuse/output等で別々のJSON serializerを作らない。

## 5. JSON Schema contract

MVPで受理するJSON Schema draftは **Draft 2020-12のみ**。

- `$schema` omitted -> Draft 2020-12として解釈
- `$schema` present -> `https://json-schema.org/draft/2020-12/schema` のみ受理
- 他Draft URIはdefinition validation error
- `jsonschema.Draft202012Validator.check_schema()` 相当でDefinition load/start前にschema自体を検証
- Runtime validationもDraft202012Validator semantics

Format validation (`format: email`, `date-time`等) はMVP既定では**annotationのみで強制しない**。親がformat checker挙動へ依存しないよう固定する。業務format検証はCustom Validatorまたは明示pattern等を使う。

## 6. トップレベル

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

Required: `name/version/jobs`。

`env`=immutable JSON-compatible literal、Expression/Secret禁止。

## 7. Workflow ID / version / hash

Workflow IDは親登録名またはWorkflowResolver canonical reference。

`version`=`1..signed64 max` integer。

Definition hash:

1. typed Definitionからruntime非依存Definition treeを作る
2. `jobrunner.canonical-json.v1`
3. SHA-256

Source YAML bytesそのものではなく意味の正規化後Definitionでhashする。Source YAML全文も別途snapshot保存。

## 8. Workflow Input

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

- `type` required
- `required` default false
- `nullable` default false
- `default` optional
- `schema` optional Draft2020-12 schema

Null:

- `nullable:true`だけがInput fieldでnull許可する方法
- caller null/default nullはnullable true必須
- required=trueはkey存在を要求。nullable=trueならnull可
- optional+default無し=missing

Validation order:

1. key presence/extra
2. nullable
3. non-nullならbase `type`
4. non-nullならoptional `schema`

Input fieldの`schema`は**non-null valueだけ**へ適用する。`nullable`をJSON Schema側の`type:[...,null]`で二重定義しない。

`schema`がbase `type`と矛盾することが静的に明白でも、最終Runtime validationでは両方満たす必要がある。推測でschemaを書き換えない。

## 9. Workflow `outputs`

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
```

Workflow success確定直前評価。失敗=`workflow_output_invalid`。

全体はname mappingによるJSON object。PayloadStore利用。

## 10. Priority / Concurrency

Priority=signed64 integer default0。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

- group final=non-empty string
- no trim/lowercase/Unicode normalization
- case-sensitive exact match
- max-runs 1..signed64 max
- on-limit queue|reject

## 11. Workflow `settings`

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
```

- max-dynamic-jobs >=0 signed64
- lease finite positive
- expiry requeue|fail
- threshold positive signed64 default4MiB

4MiBはmaximumでなくinline/spill threshold。

## 12. Job ID / Dependencies

Static Job ID:

```text
^[A-Za-z_][A-Za-z0-9_-]*$
```

`[]/` reserved for Dynamic logical key。

needs missing/self/duplicate/cycle reject。`foreach.parent`もDAG edge。

## 13. Action / Validator identity

```text
action_id/version: non-empty string
validator_id/version: non-empty string
```

Registryは親bootstrapで各Process再構築。Version update親責任。Implicit string conversion無し。

Run startで参照Action/Validator ID+version snapshot。

### Validator callable

```python
def validate_result(value, input_data) -> ValidationResult:
    ...
```

ValidationResult:

```text
valid: boolean
code: non-empty string optional
message: string optional
retryable: boolean default false
details: JSON-compatible optional
```

- read-only validation、result transform禁止
- Runtime Handle/Secret value無し
- heavy/I/O validationはnormal Job
- exception=`validator_exception`, retryable=false

## 14. Job共通field

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

- runs-on: internal only, default system pool name `default`, non-empty
- executor: internal|external_llm|human; reusable uses `uses`
- action: internal required
- validator: internal/external optional; human/reusable forbidden
- with: final Job Input object
- if: boolean CEL default success()
- success_if: internal/external only
- continue-on-error: boolean/CEL, activation snapshot
- priority: signed64
- timeout-minutes: internal only finite positive
- retry: `10`
- outputs.schema: optional Draft2020-12 schema
- external: external_llm only

## 15. Result validation order

Internal/External:

1. JSON-compatible + canonical JSON v1
2. optional Draft2020-12 Output Schema
3. optional Custom Validator
4. optional success_if
5. SecretGuard
6. PayloadStore
7. terminal success

Validator invalid:

```text
category=validation
code=validator code or domain_validation_failed
retryable=validator result
```

## 16. Executor constraints

Internal: action required; validator/runs-on/timeout optional。

External: validator/external optional; action/uses/runs-on/timeout forbidden。

Human: action/validator/uses/runs-on/success_if/external/timeout forbidden。

Reusable: uses required; action/validator/executor/runs-on/success_if/external/timeout forbidden。

## 17. Dynamic / Definition Snapshot

Dynamic syntaxは`05`。

Run snapshot:

- workflow id/version/name
- source YAML
- typed Definition JSON/hash
- Workflow Input
- Action ID/version
- Validator ID/version
- optional source_identity

## 18. 検証

Load:

- YAML1.2 safe parse
- duplicate/merge/tag
- Pydantic schema/unknown keys
- JSON Schema Draft/check_schema
- Input nullable/default
- dependency cycles
- numeric finite/range
- CEL/JMESPath compile
- executor constraints

Run start:

- Input
- Action/Validator Registry versions
- Runner Pool
- concurrency string
- Reusable resolution
- Secret policy/settings

FailureならRun row無し。

## 19. 受入条件

1. YAML1.2 booleans vs YAML1.1 ambiguity
2. canonical JSON v1 exact bytes/hash golden cases
3. 1 vs1.0 distinct digest
4. Unicode no-normalization behavior
5. Draft2020-12 only / other draft reject
6. format annotation not enforced
7. Input nullable + schema non-null application
8. dependency install Python3.10
9. Action/Validator identity
10. concurrency identity
11. numeric boundaries
12. arbitrary JSON Output/spill
13. Dynamic cycle
14. deterministic Definition hash
