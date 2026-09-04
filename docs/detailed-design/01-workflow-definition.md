# 01. Workflow Definition 詳細設計

- Status: Draft v0.8
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML、型、JSON Schema、canonical serialization、Action/Validator定義、reload、Workflow単位settingsの正規契約を定義する。

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

Definition hash、Input digest、Output digest、reuse key等で共通利用するserializationを `jobrunner.canonical-json.v1` とする。

```python
json.dumps(
    value,
    ensure_ascii=False,
    sort_keys=True,
    separators=(",", ":"),
    allow_nan=False,
)
```

結果をUTF-8 encode。

追加規則:

- object key stringのみ
- duplicate key禁止
- NaN/Infinity禁止
- datetime/Decimal/bytes/tuple等の暗黙変換禁止
- internal typed valueはJSON treeへ明示変換後canonicalize
- Unicode normalizationを暗黙適用しない
- `1`と`1.0`は異なるJSON表現

Hash/digest=`SHA-256(bytes)` lowercase hex64。

## 5. JSON Schema contract

Draft **2020-12のみ**。

- `$schema` omitted -> Draft2020-12
- present -> `https://json-schema.org/draft/2020-12/schema`のみ
- other Draft reject
- Definition load時`Draft202012Validator.check_schema()`相当
- RuntimeもDraft202012 semantics
- `format`はMVPではannotation扱いで強制しない

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

Required=`name/version/jobs`。

`env`=immutable JSON-compatible literal、Expression/Secret禁止。

## 7. Workflow ID / version / hash

Workflow ID=親登録名またはWorkflowResolver canonical reference。

`version`=1..signed64 max integer。

Definition hash:

1. typed Definition runtime非依存tree
2. canonical-json-v1
3. SHA-256

Source YAML全文も別snapshot。

## 8. Workflow Input

標準型:

```text
string/integer/number/boolean/object/array
```

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

- nullable default false
- null許可はnullable:trueのみ
- required=trueはkey存在
- optional+default無し=missing
- extra reject

Validation:

1. presence/extra
2. nullable
3. non-null base type
4. non-null optional Draft2020-12 schema

Input `schema`はnon-null valueへ適用。

## 9. Workflow `outputs`

Name -> literal/expression mapping。Workflow success確定直前評価。Failure=`workflow_output_invalid`。

Output全体object、PayloadStore利用。

## 10. Priority / Concurrency

Priority=signed64 integer default0。

Concurrency:

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

- group non-empty string
- no trim/lowercase/Unicode normalization
- case-sensitive exact
- max-runs 1..signed64 max
- on-limit queue|reject

## 11. Workflow `settings`

MVP exact keys:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
  retention:
    run-history-days: null
    execution-logs-days: null
    event-days: null
    artifact-metadata-days: null
    managed-artifact-data-days: null
```

### Runtime settings

- `max-dynamic-jobs`: integer >=0 signed64, default1000
- `external-lease-minutes`: finite positive, default60
- `external-on-lease-expiry`: requeue|fail, defaultrequeue
- `output-inline-threshold-bytes`: positive signed64, default4MiB

4MiBはinline/spill thresholdでmax Outputではない。

### Retention settings

各retention値:

```text
null = unlimited
integer >=1 = 作成/完了基準の日数
```

System configも同じ5項目を持ち、System defaultは全て`null`。

Effective policyは **Workflow setting（指定項目のみ） > System config > unlimited default**。Workflow Run開始時にeffective retention policyをsnapshotし、そのRunの後続Retentionに使用する。

- run-history-days: Workflow Run DB履歴とそれに必須従属するデータの最大保持期間
- execution-logs-days: Execution Log file
- event-days:通常Event row
- artifact-metadata-days: Artifact metadata/history
- managed-artifact-data-days: Managed ArtifactStore data

External Reference Artifactの外部実体はretention対象外。

Run historyが先に期限切れになる場合、FK整合上そのRun所有データもRun削除時に削除される。したがって各componentの実効保持期間はrun-history-daysを超えない。

`idempotency` TTLはRetention settingsと別でSystem default24h。

Unknown settings/retention key reject。

## 12. Job ID / Dependencies

Static Job ID `^[A-Za-z_][A-Za-z0-9_-]*$`。`[]/`はDynamic用予約。

needs missing/self/duplicate/cycle reject。foreach.parentもDAG edge。

## 13. Action / Validator identity

Action/Validator ID+version=non-empty string。Registry親bootstrap。Implicit conversion無し。Run start snapshot。

Validator:

```python
def validate_result(value, input_data) -> ValidationResult: ...
```

ValidationResult=`valid`, optional code/message/details, retryable defaultfalse。

Read-only、Secret/Runtime Handle無し、heavy validationはnormal Job、exception=`validator_exception`。

## 14. Job共通field

```yaml
jobs:
  example:
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

- runs-on internal only/non-empty/default pool
- action internal required
- validator internal/external optional, human/reusable forbidden
- with -> Job Input object
- if boolean default success()
- success_if internal/external
- priority signed64
- timeout internal finite positive only
- retry -> `10`
- outputs.schema Draft2020-12
- external external_llm only

## 15. Result validation order

1. JSON-compatible/canonical JSON v1
2. optional Draft2020-12 Output Schema
3. optional Custom Validator
4. optional success_if
5. SecretGuard
6. PayloadStore
7. success

## 16. Executor constraints

Internal: action required; validator/runs-on/timeout optional。

External: validator/external optional; action/uses/runs-on/timeout forbidden。

Human: action/validator/uses/runs-on/success_if/external/timeout forbidden。

Reusable: uses required; action/validator/executor/runs-on/success_if/external/timeout forbidden。

## 17. Workflow reload

Workflow Definition sourceは**親Process再起動なしでreload可能**にする。

Standard filesystem WorkflowResolver:

- `wf_definition_list/info` と `wf_start` でsource metadata (`mtime_ns + size`) を確認
- metadata変化時はfileを再read/parse/validateしてRegistry cacheをreplace
- metadata同一でも親は`WorkflowResolver.refresh(workflow_ref=None)`を明示call可能
- invalid new YAMLはcacheのold Definitionで黙って実行せず、そのreferenceをvalidation error状態として扱いnew Run startを拒否
- existing Workflow Runは自身のsnapshotを使い影響無し

File watcher/background hot reloadは必須ではない。Python Action/Validator code reloadもJobRunner専用機構を作らず親development autoreloadに任せる。

## 18. Definition Snapshot

- workflow id/version/name
- source YAML
- typed Definition JSON/hash
- Workflow Input
- Action/Validator ID+version
- effective retention policy
- optional source_identity

## 19. 検証

Load: YAML1.2、安全性、Pydantic、JSON Schema draft、Input、cycle、numeric、expression、executor/settings。

Run start: Input、Registry versions、Pool、concurrency、Reusable、Secret/settings、effective retention。

FailureならRun row無し。

## 20. 受入条件

1. YAML1.2 ambiguity
2. canonical JSON golden
3. Draft2020-12 only
4. Input nullable
5. Registry identity
6. retention settings inheritance/unlimited
7. unknown retention key reject
8. reload valid YAML without process restart
9. reload invalid YAML blocks new Runs but old Run snapshot continues
10. explicit refresh
11. concurrency identity
12. arbitrary Output/spill
13. deterministic hash
