# 06. Reusable Workflows 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`, `05-dynamic-jobs.md`

## 1. 目的

Reusable Workflow の参照、親子Workflow Run、Input/Output、Definition binding、Retry/Cancel/Recoveryを定義する。

## 2. 基本原則

1. 親から見るReusable Workflowは1 Job。
2. 子は独立Workflow Run。
3. 親子mutable stateは共有しない。
4. Input/Output/Artifactのみ明示mapping。
5. Child Definitionもsnapshotする。
6. cycleは禁止。固定depth limitは置かない。
7. Parent Job Retryでは最初に確定したChild bindingを再利用する。

## 3. YAML

```yaml
jobs:
  analyze:
    uses: ./common/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
```

`uses` Jobでは `action/executor/runs-on/success_if/external/timeout-minutes` 禁止。

## 4. `uses` reference

`uses` は literal stringのみ。

### 4.1 Relative file reference

```yaml
uses: ./common/analyze.yml
```

**基準directoryは、呼び出し元 Workflow Definition source file が存在するdirectory。**

例:

```text
/workflows/root.yml
/workflows/common/analyze.yml
```

`root.yml` から `./common/analyze.yml` は後者へ解決する。

規則:

- caller sourceがfilesystem fileである場合のみrelative fileを許可
- `.yml | .yaml`
- absolute path禁止
- canonical解決後、configured Workflow root外へ出るpathは禁止
- `..` 自体はcaller directory内の兄弟参照に必要なら許可できるが、canonical pathがWorkflow root外ならreject
- symlink解決後もWorkflow root外ならreject

### 4.2 Registered Workflow ID

```yaml
uses: common.analyze
```

`./` または `../` で始まらず `.yml/.yaml` file pathとして扱わない値はRegistry ID。

### 4.3 Non-filesystem caller

DB/Registry等からロードされ、source directoryを持たないWorkflowはrelative file `uses` を使用不可。Registered Workflow IDを使う。

URL/HTTP/GitをYAMLから直接fetchしない。

## 5. Binding

Parent Job最初のactivation時に referenceを解決し `reusable_bindings` へ固定する。

保存:

```text
parent_workflow_run_id
parent_job_run_id
workflow_ref_original
child_workflow_id
child_workflow_version
child_definition_yaml/json/hash
child_action_versions_json
created_at
```

同一Parent Job Runに1 binding。

Binding後のChild source変更はParent Retryへ反映しない。

## 6. Child Run作成

Parent Attempt開始時、bindingからChild Runを exactly one 作成する。

Atomic:

- Parent Attempt -> waiting_child
- Child Workflow Run
- parent/child relation
- Child initial Jobs
- Event

Childはresolverから再読込しない。

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

`with` をChild Input Schemaで検証。Secret参照は禁止。

Child stateは独立。親stateの直接read/writeは禁止。

ActorContext / AccessScopeは親から継承し権限拡大しない。

## 9. Workflow Output

Childトップレベル `outputs` をsuccess確定直前に評価し、Parent Job Outputとして公開する。

失敗は `workflow_output_invalid`。

## 10. Conclusion propagation

| Child | Parent Job |
| --- | --- |
| success | success |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Parent `continue-on-error` は通常規則で適用する。

## 11. Retry

Parent Retry:

- new Parent Attempt
- same reusable binding
- new Child Workflow Run
- same Parent Input

BindingされたAction versionが現在Registryに無ければ `action_version_mismatch`。新Childへ自動upgradeしない。

## 12. Cancel / Pause

Parent cancelはcurrent Childへ伝播。

Parent Pauseはstarted Childへ伝播しない。Parent Pause中はnew Child Run開始禁止。

MVPではChild Workflow Runへのpublic direct:

```text
pause/resume/cancel/retry/priority update
```

を禁止し `child_run_direct_control_forbidden`。

read/info/log/artifactは認可範囲内で許可。

## 13. Cycle / depth

canonical `workflow_id` ancestor chainでcycle検出。

固定depth limit無し。`call_depth`は保存する。

## 14. Dynamic Job

Reusable JobにRoot/Nested `foreach`を付けられる。Generated Jobごとにbinding/Child Runを持つ。

Parent Run generated Job上限はParent側、Child generated JobはChild Run側で別カウント。

## 15. Child concurrency

Childは自身のWorkflow concurrencyを持つ。Parent groupは自動継承しない。

Child concurrency wait中、Parent Jobは `waiting_child`。

## 16. Recovery / uniqueness

Runtime再起動後:

- binding/relation復元
- Child runningなら通常Recovery
- Child completedならParentへidempotent propagation
- same Parent AttemptへChild重複作成禁止

DB保証:

- one binding / parent_job_run_id
- one Child Run / parent_attempt_id

## 17. Failure code

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
child_run_direct_control_forbidden
```

## 18. 受入条件

1. caller-directory基準relative reference
2. root traversal/symlink escape拒否
3. non-filesystem callerのrelative拒否
4. registered ID
5. binding固定Retry
6. Child state isolation
7. cycle検出
8. direct Child control拒否
9. Dynamic + Reusable
10. restart duplicate防止
