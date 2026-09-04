# 01. Workflow Definition 詳細設計

- Status: Draft v1.5
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML、厳密型、JSON Schema、canonical serialization、Action/Validator定義、設定継承、priority、concurrency、reload、Job fieldの正規契約を定義する。

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

## 3. YAML / typed model基本原則

1. Canonical authoring format=YAML 1.2。
2. `ruamel.yaml` safe loaderをYAML 1.2 modeで使用。
3. custom tag/任意object construction禁止。
4. duplicate mapping key reject。
5. merge key `<<` reject。
6. unknown key reject。
7. load後typed immutable `WorkflowDefinition`。
8. Core typed modelは**strict / no coercion**。
9. 数値NaN/Infinity reject。
10. SQLite INTEGER対象はsigned64範囲。
11. Run開始時はsource bytesを再読込・再検証し実使用Definition snapshot固定。

YAML 1.1の暗黙boolean等へfallbackしない。

### 3.1 JSON-compatible strict type semantics

CoreでJSON値を型判定する正規規則:

```text
null     -> None only
boolean  -> bool only
integer  -> Python int, but bool excluded
number   -> int or float, but bool excluded and finite only
string   -> str only
object   -> mapping/object with string keys only
array    -> list/JSON array only
```

- `"1"` をintegerへ変換しない
- `1` をstringへ変換しない
- `true` をinteger/numberとして扱わない
- `1`はintegerかつnumberとして許可され得るが、`1.0`はintegerではない
- tuple/set/Decimal/datetime/bytes等をJSON型へ暗黙変換しない
- Default値も同じstrict規則でDefinition load時に検証
- Pydantic modelはstrict設定または同等の明示validatorでこの規則を保証する

HTTP query/pathの文字列を数値やbooleanへ変換する責務はAdapterにあり、Core Service modelへ渡る時点では正規型にする。Exact Adapter規則=`11`。

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
- MVPのSchemaは**自己完結型**とし、CoreはHTTP/file/parent filesystem等からSchemaを取得しない
- `$defs`等による同一Schema document内の再利用は許可
- `$ref` / `$dynamicRef` は `#` で始まるlocal fragmentだけ許可
- absolute URI、relative URI、file path等のnon-fragment `$ref` / `$dynamicRef` はDefinition load時にreject
- Resolver/network policyへ処理を委ねて外部Schemaを黙って取得する実装は禁止

外部参照禁止違反は `json_schema_external_ref_forbidden` としてDefinition validation errorにする。

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

`version`=1..signed64 max integer。bool/string等のcoercion無し。

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
- base typeは§3.1 strict semantics
- Defaultも同じ型/nullable/schemaをDefinition load時に満たすこと

Validation:

1. presence/extra
2. nullable
3. non-null strict base type
4. non-null optional Draft2020-12 schema

Input `schema`はnon-null valueへ適用。

## 9. Workflow `outputs`

Name -> literal/expression mapping。Workflow success確定直前評価。Failure=`workflow_output_invalid`。

Output全体object、PayloadStore利用。

## 10. Priority / Concurrency

Workflow Definition `priority`=signed64 integer、default0。bool/string coercion無し。

Root Workflow Run初期priority:

```text
wf_start.priority specified -> request value
otherwise -> Workflow Definition priority
```

Reusable Child RunはChild Definitionのtop-level priorityを使わずroot Workflow Runのcurrent priorityを継承する。Root `wf_priority_update` はroot自身と全non-terminal descendant Child Runへ同値を伝播し、future Childも更新後root priorityを継承する。Child direct update禁止。

Job `priority` はsigned64 integer、default0。

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
- max-runs 1..signed64 max integer、bool/string coercion無し
- on-limit queue|reject
- scope key=`(workflow_id, resolved group)`
- 別Workflow IDの同じgroup文字列は競合しない
- 同じWorkflow ID/groupのactive Runだけをmax-runs対象
- active holder/candidateのmax-runs差は`08` conservative capacity規則

## 11. System Workflow Defaults / Workflow `settings`

System config canonical defaults:

```text
default_runner_pool = "default"
max_dynamic_jobs = 1000
external_lease_minutes = 60
external_on_lease_expiry = requeue
output_inline_threshold_bytes = 4194304
execution_log_level = normal
workflow_progress_mode = auto
job_progress_mode = auto
idempotency_ttl_hours = 24
retention.* = null
```

Workflow YAML:

```yaml
settings:
  max-dynamic-jobs: 1000
  external-lease-minutes: 60
  external-on-lease-expiry: requeue
  output-inline-threshold-bytes: 4194304
  execution-log-level: normal
  workflow-progress-mode: auto
  job-progress-mode: auto
  retention:
    run-history-days: null
    execution-logs-days: null
    event-days: null
    artifact-metadata-days: null
    managed-artifact-data-days: null
```

### 11.1 Root Run system baseline snapshot

Root Workflow Run開始時にWorkflow実行へ影響するSystem baselineを `system_workflow_defaults_json` としてsnapshotする。

Exact shape:

```text
default_runner_pool
max_dynamic_jobs
external_lease_minutes
external_on_lease_expiry
output_inline_threshold_bytes
execution_log_level
workflow_progress_mode
job_progress_mode
retention:
  run_history_days
  execution_logs_days
  event_days
  artifact_metadata_days
  managed_artifact_data_days
```

`idempotency_ttl_hours` はRun execution semanticsではないためsnapshot外。

Root effective settings/RetentionはSystem baseline snapshot + Workflow settingsから算出。

Reusable Childはcurrent System configを読み直さずParent Run `system_workflow_defaults_json` を継承してChild settings/Retentionを算出する。

### 11.2 Effective runtime setting

```text
Workflow specified value > Run system baseline snapshot > canonical default
```

`default_runner_pool`はSystem-onlyでWorkflow YAMLから上書きしない。ただしRun effective settingsへcopyする。

External Job lease/expiryは§18 Job override最優先。

Validation:

- `default_runner_pool`: non-empty string
- `max-dynamic-jobs`: strict integer >=0 signed64
- `external-lease-minutes`: finite positive number, bool/string不可
- `external-on-lease-expiry`: requeue|fail
- `output-inline-threshold-bytes`: strict positive signed64 integer
- `execution-log-level`: normal|debug
- `workflow-progress-mode`: auto|none
- `job-progress-mode`: auto|explicit|none

4MiBはinline/spill thresholdでmax Outputではない。

`idempotency_ttl_hours`はSystem configのみ、finite positive number。

### 11.3 Retention settings

各retention値:

```text
null = unlimited
strict integer >=1 = 日数
```

Effective policy=Workflow指定 > Run system baseline retention > unlimited default。

- run-history-days
- execution-logs-days
- event-days
- artifact-metadata-days
- managed-artifact-data-days

External Reference Artifactの外部実体はretention対象外。Run history expiryはowned component最終上限。

Unknown settings/retention key reject。

## 12. Job ID / Dependencies

Static Job ID `^[A-Za-z_][A-Za-z0-9_-]*$`。`[]/`はDynamic用予約。

`needs` missing/self/duplicate/cycle reject。`foreach.parent`もDAG edge。

## 13. Action / Validator Registry identity

YAMLはAction/Validator IDだけを指定し、versionは親RegistryからRun start時に解決する。

MVP Registry:

```text
action_id -> exactly one current {version, callable, uses_runtime, metadata}
validator_id -> exactly one current {version, callable}
```

- ID/version=non-empty string
- `uses_runtime` strict boolean、default false
- 同一ID二重登録reject
- Multi-version Registry無し
- Run start時current version snapshot
- Retry/Resume/Runner executionはsnapshot versionとcurrent Registry version exact一致
- `uses_runtime`や実装を同じversionのまま変えた場合は親責任

Action invocation=`04`。CoreはsignatureからRuntime Handle要否を推測しない。

ValidatorはMVPでは同期・軽量 callable:

```python
def validate_result(value, input_data) -> ValidationResult: ...
```

ValidationResult=`valid`, optional code/message/details, retryable defaultfalse。Heavy/async validationはnormal Job。

## 14. Job共通field

Canonical shape:

```yaml
jobs:
  example:
    needs: []
    executor: internal
    runs-on: default
    action: system.example
    validator: domain.validate_example
    uses: null
    with: {}
    if: ${{ success() }}
    success_if: ${{ outputs.ok == true }}
    continue-on-error: false
    priority: 0
    timeout-minutes: 30
    retry: {}
    outputs:
      schema: null
    progress:
      mode: auto
    foreach: null
    key: null
    order_by: null
    external: null
```

Unknown Job field reject。全typed fieldは§3.1/field-specific strict type規則を適用し、文字列/boolean/numberの暗黙変換をしない。

### 14.1 `executor` resolution

YAMLで明示可能=`internal|external_llm|human`。

1. `uses` present -> executor省略必須、resolved=`reusable`
2. `uses` absent + executor omitted -> `internal`
3. explicit ->その値

YAMLで`executor: reusable`は書かない。

### 14.2 `runs-on` resolution

Internal Jobだけ。

- explicit non-empty string ->そのPool
- omitted -> Run `effective_settings.default_runner_pool`
- resolved PoolはJob Run/Generated Job作成時snapshot
- 未登録PoolはRun start/Dynamic expansion preflightでfail-closed

### 14.3 Job `outputs`

```yaml
outputs:
  schema:
    type: object
```

のみ。Omitted/`{}`=Schema無し。Unknown key reject。

Humanは`outputs.schema`禁止。Reusableはoptional、Child Workflow Output objectへ適用。

### 14.4 Job `progress`

```yaml
progress:
  mode: auto|explicit|none
```

Omitted -> Workflow `settings.job-progress-mode` -> Run baseline/default auto。Exact semantics=`09`。

## 15. Result validation order

Internal/External:

1. JSON-compatible/canonical JSON v1
2. optional Draft2020-12 Output Schema
3. optional Custom Validator
4. optional success_if
5. SecretGuard
6. PayloadStore
7. success

Reusable:

1. Child success + Child Workflow Output object
2. optional Parent Job `outputs.schema`
3. SecretGuard
4. Parent Attempt PayloadStore
5. success

Human approve Output=`null`、Schema/Validator/success_if無し。

## 16. Executor constraints

Internal:

- action required
- validator/runs-on/timeout optional
- uses/external forbidden

External:

- validator/external optional
- action/uses/runs-on/timeout forbidden

Human:

- action/validator/uses/runs-on/success_if/external/timeout forbidden
- non-empty `outputs.schema` forbidden

Reusable:

- uses required
- action/validator/executor/runs-on/success_if/external/timeout forbidden
- `outputs.schema` optional

`with/if/continue-on-error/priority/retry/progress/foreach/key/order_by` は共通規則に従う。

## 17. Secret expression field restriction

`${{ secrets.NAME }}` はinternal Job `with`だけ、かつ1 scalar全体のみ。

Persistent表現/Result Reuse制約=`02/03/12`。

## 18. External Job override

External Jobのみ:

```yaml
external:
  lease-minutes: 120
  on-lease-expiry: requeue
```

Allowed keys=この2つ。

Effective=`Job external value > Workflow Run effective setting`。

- lease-minutes finite positive number、bool/string coercion無し
- on-lease-expiry requeue|fail

## 19. Workflow Resolver / reload

Workflow Definition sourceは親Process再起動なしでreload可能。

### 19.1 Browse/info cache

Standard filesystem Resolverは`wf_definition_list/info`向けに `mtime_ns + size` 等のmetadata cacheを使ってよい。

- metadata変化時はfile再read/parse/validate/hashしてcache replace
- `WorkflowResolver.refresh(workflow_ref=None)` で明示refresh可能
- invalid sourceはそのread/info requestでvalidation errorにできる

### 19.2 Execution-time source read

**新しい実行を開始/新Reusable bindingを作るときはmetadata cacheだけをSource of Truthにしない。**

以下では対象Workflow source bytesをその時点で必ず1回readし、そのbytesを直接parse/validate/hashする。

```text
wf_start
Reusable binding first creation
```

- mtime/sizeがcacheと同じでもexecution pathではsource bytesを再readする
- readした同一bytesからsource YAML snapshot + typed Definition + Definition hashを作る
- validation失敗時はold cacheへsilent fallbackせず開始拒否
- read途中のfile replacement対策としてResolverは1回のlogical readで一貫したbytesを取得する実装を使う（open/read/closeしたbytesがその開始処理のSource of Truth）
- Existing Run/既存Reusable bindingは保存済みsnapshotを使用しsourceを再読込しない

File watcher/background hot reload必須無し。Python Action/Validator code reloadは親development autoreloadへ任せる。

## 20. Definition / Run Snapshot

Definition snapshot:

- workflow id/version/name
- execution pathで実読込したsource YAML
- typed Definition JSON/hash
- Workflow Input
- Action/Validator ID+version

Root Run additionally:

- `system_workflow_defaults_json`
- effective runtime settings
- effective retention policy
- initial priority
- optional source_identity

Child Run snapshot rules=`06`。

## 21. 検証

Load:

- YAML1.2、安全性、strict Pydantic/validators
- JSON Schema Draft + self-contained local-reference policy
- Input/default strict types
- needs/foreach cycle
- numeric/expression
- executor/field conflicts
- settings/external/progress
- Secret placement
- Human/Reusable outputs.schema constraint

Run start:

- fresh source bytes read + parse/validate/hash
- Input strict validation
- current Registry versions
- System workflow defaults snapshot
- default/explicit Runner Pool
- priority resolution
- concurrency scope/admission
- Reusable refs
- effective settings/retention
- Authorization

FailureならRun row無し。

## 22. 受入条件

1. YAML1.2 ambiguity
2. strict/no-coercion Core model
3. bool not integer/number
4. string numeric not coerced
5. Input/default strict validation
6. canonical JSON golden
7. Draft2020-12 only
8. JSON Schema local fragment `$ref/$dynamicRef` allow + external/relative ref reject + no retrieval
9. executor default/internal + uses->reusable
10. Root System baseline snapshot/restart stability
11. Child Parent baseline inheritance
12. default_runner_pool/runs-on resolution
13. root priority resolution/Child propagation
14. concurrency scope=(workflow_id,group)
15. mixed max-runs delegates to `08`
16. Job outputs.schema Human/Reusable boundary
17. Registry one-current-version/uses_runtime/version mismatch
18. runtime setting/External lease/progress/log/Retention strict validation
19. Secret full-scalar
20. browse cache refresh
21. wf_start always reads source bytes even unchanged metadata
22. Reusable first binding always reads child source bytes
23. invalid execution-time source never falls back to old cache
24. arbitrary Output/spill
25. deterministic Definition hash
