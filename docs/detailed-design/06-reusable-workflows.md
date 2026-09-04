# 06. Reusable Workflows 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `05`, `08`, `10`

## 1. 目的

Reusable Workflow の参照、親子Workflow Run、Input/Output、Definition binding、Action/Validator version、Retry/Cancel/Recoveryを定義する。

## 2. 基本原則

1. 親から見るReusable Workflowは1 Job。
2. 子は独立Workflow Run。
3. 親子mutable stateは共有しない。
4. Input/Output/Artifactのみ明示mapping。
5. Child Definitionもsnapshotする。
6. cycleは禁止。固定depth limitは置かない。
7. Parent Job Retryでは最初に確定したChild bindingを再利用する。
8. BindingはChildのAction versionだけでなくValidator versionも固定する。

## 3. YAML

```yaml
jobs:
  analyze:
    uses: ./common/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
```

`uses` Jobでは `action/validator/executor/runs-on/success_if/external/timeout-minutes` 禁止。

## 4. `uses` reference

`uses` は literal stringのみ。

### 4.1 Relative file reference

```yaml
uses: ./common/analyze.yml
```

基準directoryはcaller Workflow Definition source fileのdirectory。

規則:

- caller sourceがfilesystem fileの場合のみrelative可
- `.yml|.yaml`
- absolute path禁止
- `..`はcanonical resultがWorkflow root内なら許可
- symlink解決後root外ならreject

### 4.2 Registered Workflow ID

`./`/`../`で始まらずfile path扱いでないliteralはRegistry ID。

### 4.3 Non-filesystem caller

source directory無しならrelative file不可。Registered IDを使う。

URL/HTTP/GitをYAMLから直接fetchしない。

## 5. Binding

Parent Job最初のactivation時にreferenceを解決し `reusable_bindings` へ固定する。

保存:

```text
parent_workflow_run_id
parent_job_run_id
workflow_ref_original
child_workflow_id
child_workflow_version
child_definition_yaml/json/hash
child_action_versions_json
child_validator_versions_json
created_at
```

同一Parent Job Runに1 binding。

### 5.1 Binding validation

1. reference resolve
2. canonical child_workflow_id
3. ancestor cycle check
4. Child Definition/Input-independent static validation
5. Child Action Registry ID/version resolve
6. Child Validator Registry ID/version resolve
7. binding persist

Action/Validator versionはnon-empty string。

Binding後のChild source/Registry mapping変更はParent Retryへ自動反映しない。

## 6. Child Run作成

Parent Attempt開始時、bindingからChild Runをexactly one作成。

Atomic:

- Parent Attempt -> waiting_child
- Child Workflow Run
- parent/child relation
- Child initial Jobs
- Event

ChildはresolverからDefinitionを再読込せずbinding snapshotを使う。

## 7. Parent/Child identity

Child Run:

```text
parent_workflow_run_id
parent_job_run_id
parent_attempt_id
root_workflow_run_id
call_depth
reusable_binding_id
```

Rootはparent null, root=self, depth=0。

## 8. Input / State / Actor

`with`をChild Input Schemaで検証。Secret参照は禁止。

Child state独立。親state直接read/write禁止。

ActorContext/AccessScopeは親から継承し権限拡大禁止。

## 9. Workflow Output

Childトップレベル`outputs`をsuccess確定直前に評価し、JSON objectとしてParent Job Outputへ公開。

失敗 `workflow_output_invalid`。

PayloadStore inline/spill規則は通常Workflowと同じ。

## 10. Conclusion propagation

| Child | Parent Job |
| --- | --- |
| success | success |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Parent `continue-on-error` は通常規則。

## 11. Retry

Parent Retry:

- new Parent Attempt
- same reusable binding
- new Child Workflow Run
- same Parent persistent Input

Retry開始前にcurrent Registryがbinding snapshotの全Action/Validator versionを提供できることを確認する。

不足:

```text
action_version_mismatch
validator_version_mismatch
```

いずれもretryable=false。新Child Definition/Validatorへ自動upgradeしない。

## 12. Result Reuse identity

Reusable Parent Jobのexecutor identityは:

```text
child_definition_hash
child_action_versions_json
child_validator_versions_json
```

を含む。Child Validator version変更を古い成功結果のsame-run reuseから隠さない。

## 13. Cancel / Pause

Parent cancelはcurrent Childへ伝播。

Parent Pauseはstarted Childへ伝播しない。Pause中new Child Run開始禁止。

MVP Child Workflow Runへのpublic direct:

```text
pause/resume/cancel/retry/priority update
```

は禁止し `child_run_direct_control_forbidden`。

read/info/log/artifactは認可範囲内で許可。

## 14. Cycle / depth

canonical `workflow_id` ancestor chainでcycle検出。固定depth limit無し。`call_depth`保存。

## 15. Dynamic Job

Reusable JobにRoot/Nested `foreach`可。Generated Jobごとにbinding/Child Run。

Parent generated上限はParent Run、Child内はChild Run自身。

## 16. Child concurrency

Childは自身のWorkflow concurrency。Parent group自動継承無し。

Child concurrency wait中Parent Job=`waiting_child`。

## 17. Recovery / uniqueness

Restart後:

- binding/relation復元
- bindingのAction/Validator version availability確認
- Child runningは通常Recovery
- Child completedはParentへidempotent propagation
- same Parent AttemptへChild重複作成禁止

DB保証:

- one binding / parent_job_run_id
- one Child Run / parent_attempt_id

## 18. Failure code

```text
workflow_not_found
workflow_reference_invalid
workflow_relative_reference_unavailable
workflow_cycle_detected
workflow_input_invalid
child_workflow_start_failed
child_workflow_failed
workflow_output_invalid
action_version_mismatch
validator_version_mismatch
child_run_direct_control_forbidden
```

## 19. 受入条件

1. caller-directory relative reference
2. root/symlink escape reject
3. non-filesystem relative reject
4. registered ID
5. binding Action+Validator versions snapshot
6. Retry same binding / missing version fail-closed
7. Reuse identity includes Child validator versions
8. Child state isolation
9. cycle
10. direct Child control reject
11. Dynamic+Reusable
12. restart duplicate防止
