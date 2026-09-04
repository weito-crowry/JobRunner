# 01. Workflow Definition 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`

## 1. 目的

JobRunner の Workflow YAML、型、JSON Schema、canonical serialization、Action/Validator定義、設定継承、priority、concurrency、reload、Job fieldの正規契約を定義する。

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

Workflow Definition `priority`=signed64 integer、default0。

Root Workflow Run初期priority:

```text
wf_start.priority specified -> request value
otherwise -> Workflow Definition priority
```

Reusable Child RunはChild Definitionのtop-level priorityを使わず、**root Workflow Runのcurrent priorityを継承**する。Root `wf_priority_update` はroot自身と全non-terminal descendant Child Runへ同値を伝播し、future Childも更新後root priorityを継承する。Childへのdirect priority updateは`06/11`どおり禁止。

Job `priority` は各Job Definitionのsigned64 integer、default0。Workflow Run priorityとは別軸。

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
- scope key=`(workflow_id, resolved group)`
- **別Workflow IDの同じgroup文字列は競合しない**
- 同じWorkflow ID/groupのactive Runだけをmax-runs対象にする
- Definition更新でactive Runとcandidateのmax-runsが異なる場合は`08`のconservative capacity規則を使う

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

Root Workflow Run開始時に、Workflow実行へ影響するSystem baselineを `system_workflow_defaults_json` としてsnapshotする。

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

`idempotency_ttl_hours` はService requestのTTLでRun execution semanticsではないため、このsnapshotには含めない。

Root Run effective runtime settings/RetentionはこのSystem baseline snapshot + Workflow Definition settingsから算出する。

Reusable Childはbinding時のcurrent System configを読み直さず、Parent Workflow Runの `system_workflow_defaults_json` を継承してChild settings/Retentionを算出する。Exact rules=`06`。

### 11.2 Effective runtime setting

```text
Workflow specified value > Run system baseline snapshot > canonical default
```

`default_runner_pool`はSystem-onlyでWorkflow YAMLから上書きしない。ただしRun effective settingsへcopyする。

External Job lease/expiryは§18 Job overrideが最優先。

Validation:

- `default_runner_pool`: non-empty string、Run start時にregistered Pool存在を要求
- `max-dynamic-jobs`: integer >=0 signed64
- `external-lease-minutes`: finite positive number
- `external-on-lease-expiry`: requeue|fail
- `output-inline-threshold-bytes`: positive signed64 integer
- `execution-log-level`: normal|debug
- `workflow-progress-mode`: auto|none
- `job-progress-mode`: auto|explicit|none

4MiBはinline/spill thresholdでmax Outputではない。

`idempotency_ttl_hours`はSystem configのみ。Workflowから上書きしない。finite positive number。

### 11.3 Retention settings

各retention値:

```text
null = unlimited
integer >=1 = 作成/完了基準の日数
```

Effective policyは **Workflow setting（指定項目のみ） > Run system baseline retention > unlimited default**。Workflow Run開始時にeffective retention policyをsnapshotする。

- run-history-days: Workflow Run DB履歴と必須従属data
- execution-logs-days: Execution Log file
- event-days: 通常Event row
- artifact-metadata-days: Artifact metadata/history
- managed-artifact-data-days: Managed ArtifactStore data

External Reference Artifactの外部実体はretention対象外。

Run history expiryは各owned componentの最終保持上限。

Unknown settings/retention key reject。

## 12. Job ID / Dependencies

Static Job ID `^[A-Za-z_][A-Za-z0-9_-]*$`。`[]/`はDynamic用予約。

`needs` missing/self/duplicate/cycle reject。`foreach.parent`もDAG edge。

## 13. Action / Validator Registry identity

YAMLはAction/Validator **IDだけ**を指定し、versionは親RegistryからRun start時に解決する。

MVP Registryは各Processで:

```text
action_id -> exactly one current {version, callable, uses_runtime, metadata}
validator_id -> exactly one current {version, callable}
```

- ID/versionはnon-empty string
- `uses_runtime` boolean、default false
- 同一Processで同じIDの二重登録はreject
- Coreは同一IDの複数historical callableを自動保持しない
- Run start時にcurrent versionをsnapshot
- Retry/Resume/Runner executionはsnapshot versionとcurrent Registry versionのexact一致を要求
- `uses_runtime`や実装を同じversionのまま変更した場合は親責任
- version不一致は`action_version_mismatch|validator_version_mismatch`

Action invocation exact contract=`04`。CoreはsignatureからRuntime Handle要否を推測しない。

同時に旧/new implementationを提供したい親は別Action/Validator IDを使う。Multi-version RegistryはMVP外。

ValidatorはMVPでは同期・軽量 callable:

```python
def validate_result(value, input_data) -> ValidationResult: ...
```

ValidationResult=`valid`, optional code/message/details, retryable defaultfalse。

Read-only、Secret/Runtime Handle無し、heavy/async validationはnormal Job、exception=`validator_exception`。

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

Unknown Job field reject。

### 14.1 `executor` resolution

YAML `executor`で明示可能なのは:

```text
internal|external_llm|human
```

Resolution:

1. `uses` present -> `executor` fieldは省略必須、resolved executor=`reusable`
2. `uses` absent + `executor` omitted -> resolved executor=`internal`
3. `uses` absent + explicit executor ->その値

YAMLで`executor: reusable`は書かない。

### 14.2 `runs-on` resolution

Internal Jobだけで使用。

- explicit non-empty `runs-on` ->そのPool名
- omitted -> Workflow Run `effective_settings.default_runner_pool`
- resolved PoolはJob Run/Generated Job Run作成時に`runs_on`へsnapshot
- 未登録PoolはRun startまたはDynamic expansion preflightでfail-closed

### 14.3 Job `outputs`

Job result schema設定:

```yaml
outputs:
  schema:
    type: object
```

のみ。Omitted/`{}`=Schema無し。Unknown key reject。

Schema=Draft2020-12。Job Output本体は任意JSON value。トップレベルWorkflow outputs name mappingとは別概念。

Human JobはOutputが常に`null`でReview metadataは別APIにあるため、Humanでは`outputs.schema`を禁止する。Reusable Jobでは`outputs.schema`を許可し、Child Workflow Output objectをParent Job Outputとして受け取った後にSchema検証する。Exact semantics=`06/07`。

### 14.4 Job `progress`

```yaml
progress:
  mode: auto|explicit|none
```

Omitted -> Workflow `settings.job-progress-mode` -> Run system baseline/default auto。Exact semantics=`09`。

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

1. Child Workflow success + Child Workflow Output object取得
2. optional Parent Job `outputs.schema`
3. SecretGuard
4. Parent Attempt PayloadStore
5. Parent Job success

Human: approve時Output=`null`をParent Attempt Outputとして保存し、Schema/Validator/success_ifは持たない。

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

`with/if/continue-on-error/priority/retry/progress/foreach/key/order_by` は共通規則に従い利用可能。

## 17. Secret expression field restriction

`${{ secrets.NAME }}` はinternal Job `with`だけ、かつ1 scalar全体のみ。

Allowed:

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

Rejected:

```yaml
with:
  auth: "Bearer ${{ secrets.API_TOKEN }}"
```

加工はAction内部。Persistent表現=`02/08/12`。

## 18. External Job override

External Jobのみ:

```yaml
external:
  lease-minutes: 120
  on-lease-expiry: requeue
```

Allowed keysはこの2つだけ。

Effective:

```text
Job external value > Workflow Run effective setting
```

Workflow Run effective settingはWorkflow setting > Run system baseline > canonical defaultで既にsnapshot済み。

- lease-minutes finite positive
- on-lease-expiry requeue|fail

## 19. Workflow reload

Workflow Definition sourceは親Process再起動なしでreload可能。

Standard filesystem WorkflowResolver:

- `wf_definition_list/info` と `wf_start` でsource metadata (`mtime_ns + size`) を確認
- metadata変化時file再read/parse/validateしてcache replace
- metadata同一でも`WorkflowResolver.refresh(workflow_ref=None)`可
- invalid new YAMLはold Definitionへsilent fallbackせずnew Run start拒否
- existing Runは自身のsnapshot継続

File watcher/background hot reload必須無し。Python Action/Validator code reloadは親development autoreloadへ任せる。

## 20. Definition / Run Snapshot

Definition snapshot:

- workflow id/version/name
- source YAML
- typed Definition JSON/hash
- Workflow Input
- Action/Validator ID+version

Root Run additionally snapshots:

- `system_workflow_defaults_json`
- effective runtime settings（default_runner_pool含む）
- effective retention policy
- initial Workflow Run priority
- optional source_identity

Child Run snapshot rules=`06`。

## 21. 検証

Load:

- YAML1.2、安全性、Pydantic
- JSON Schema draft
- Input
- needs/foreach cycle
- numeric/expression
- executor/field conflicts
- settings/external/progress
- Secret placement/full-scalar
- Human/Reusable outputs.schema constraint

Run start:

- Input
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
2. canonical JSON golden
3. Draft2020-12 only
4. Input nullable
5. executor default/internal + uses->reusable
6. Root System baseline snapshot/restart stability
7. Child uses Parent Run System baseline rather than current System config
8. default_runner_pool snapshot/runs-on resolution
9. root priority request override/definition default
10. Child priority inheritance/root update propagation
11. concurrency scope=(workflow_id,group)
12. mixed max-runs delegates to `08` conservative admission
13. Job outputs.schema exact shape + Human/Reusable boundary
14. Registry one-current-version/uses_runtime metadata/version mismatch
15. runtime setting inheritance
16. External lease hierarchy
17. progress/log setting validation
18. Secret full-scalar
19. retention inheritance
20. reload valid/invalid/refresh
21. arbitrary Output/spill
22. deterministic Definition hash
