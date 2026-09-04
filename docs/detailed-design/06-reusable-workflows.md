# 06. Reusable Workflows 詳細設計

- Status: Draft v0.2
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
5. 子Definitionもsnapshotする。
6. Cycleは禁止。固定depth limitはMVP必須にしない。
7. 親Job Retryでは**最初に確定したChild Definition bindingを再利用**し、変更後のChild YAMLを取り直さない。

## 3. YAML

```yaml
jobs:
  analyze:
    uses: ./workflows/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
```

`uses` Jobでは `action/executor/runs-on/success_if/external` 禁止。

Dynamic `foreach`と組み合わせ可能。

## 4. `uses` reference 正規syntax

`uses`はliteral stringのみ。expressionは禁止。

### relative file

```yaml
uses: ./workflows/common/analyze.yml
```

- `./`で始まる
- extension `.yml | .yaml`
- configured Workflow root内へ正規化する
- absolute path禁止
- root外へ出る`..` traversal禁止
- symlink解決後もroot外なら拒否

### registered Workflow ID

```yaml
uses: common.analyze
```

`./`で始まらない値はWorkflow Registry IDとして扱う。

URL/HTTP/Git referenceをYAMLから直接fetchしない。

`WorkflowResolver`はどちらもcanonical `workflow_id`へ解決してcycle検出・bindingに使う。

## 5. Parent Job Input

`with`をChild Workflow Inputへ解決し、Child Input Schemaで検証する。

Secret referenceは禁止。ChildがSecretを必要とする場合Child自身のinternal Action JobがSecret Providerを使う。

Input invalid時はChild Runを作らずParent Attempt failure。

## 6. Reusable Binding

Parent Jobの**最初のactivation時**にChild referenceを解決し、`reusable_bindings`へ固定する。

保存:

```text
binding_id
parent_workflow_run_id
parent_job_run_id
workflow_ref_original
child_workflow_id
child_workflow_version
child_definition_yaml
child_definition_hash
child_action_versions_json
created_at
```

### 6.1 binding作成

1. `uses`解決
2. canonical workflow_id取得
3. ancestor cycle検証
4. Child Definition static/runtime validation
5. Child Action versions解決
6. bindingを1回だけ保存

同一Parent Job Runにbindingは1件。

### 6.2 source変更

Parent Run開始からParent Job初activationまでにChild sourceが変わった場合、初activation時の現行Childがbindingされる。**binding後の変更はParent Job Retryに影響しない。**

これはParent snapshotにChild全体を前もって埋め込まず、実際に呼ばれた時点で独立Child Runを固定するための規則。

## 7. Child Workflow Run作成

Parent Attempt開始時、bindingからChild Runを作成する。

同一Parent AttemptにChild Runはexactly one。

Atomic保存:

- Parent Attempt / waiting_child
- Child Workflow Run
- parent/child relation
- Child initial Jobs
- Events

Childはresolverから再読込せずbindingのDefinition/Action versionsを使用する。

## 8. relation

Child Workflow Run:

```text
parent_workflow_run_id
parent_job_run_id
parent_attempt_id
root_workflow_run_id
call_depth
reusable_binding_id
```

Root Runはparent fields null、`root_workflow_run_id=self`、`call_depth=0`。

## 9. Child Snapshot

通常Run同様:

- binding済みChild YAML/hash/version
- Child Input
- Child Action versions
- optional source_identity

をChild Runへ保存する。

## 10. State / Actor / Scope

Child stateは独立namespace。親state直接read/writeは禁止。

ActorContext / AccessScopeは親から継承し、権限拡大しない。Child側で狭めることは可能。

## 11. Workflow Output

Child YAMLのトップレベル:

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
  report: ${{ jobs.report.artifacts.report }}
```

`jobs` contextは`02`のWorkflow Output用shape。

Child success確定直前にOutput mappingを評価しJSON-compatible objectへまとめる。評価失敗はChild `failure/workflow_output_invalid`。

Parent Job success時、そのobjectをParent Job Outputとして公開する。

Artifact referenceも同じOutput object内またはParent Job Artifact mappingとして扱える。

## 12. Conclusion propagation

| Child | Parent Job |
| --- | --- |
| success | success |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Parent Jobの`continue-on-error`はこのParent conclusionへ通常規則として適用される。

## 13. Retry

Parent failed Job manual/auto Retry:

- new Parent Attempt
- **既存Reusable Bindingを使用**
- new Child Workflow Run
- Parent Input同一

過去Child Runは履歴として保持する。

```text
Parent Job
├─ Attempt 1 -> Child A failure
└─ Attempt 2 -> Child B success
```

Child source/action versionがbinding後に変わり、現在RegistryがbindingされたAction versionを提供できない場合は`action_version_mismatch`でfail-closed。新Child Definitionへ自動upgradeしない。

## 14. Cancel

Parent Workflow Run/Parent Job cancelはcurrent Childへ伝播。

Child cancelは通常graceful cancel。

### public direct control

MVPでは`parent_workflow_run_id != null`のChild Workflow Runへのpublic:

```text
pause
resume
cancel
retry
priority update
```

を拒否する。

code:

```text
child_run_direct_control_forbidden
```

read/info/log/artifact参照はAuthorization範囲内で許可。

理由はParent Job stateとの不整合を避けるため。内部のParent propagation操作は許可する。

## 15. Pause

Parent Pauseは**開始済みChildへ伝播しない**。Child内running/queued JobはChild自身のSchedulingに従って進行できる。

ただしParent Pause中は新しいParent Job / new Child Runを開始しない。

Child自身をpublic pauseするAPIはMVPでは禁止（前節）。

## 16. Cycle / depth

禁止:

```text
A -> A
A -> B -> A
A -> B -> C -> A
```

canonical workflow_idのancestor chainでChild開始前に検査。`call_depth`保存。

fixed depth limitは置かないが、cycle無しでもOS/DB資源を無限利用できるわけではなく各WorkflowのDynamic limit等は別途適用される。

## 17. Dynamic Jobとの組合せ

```yaml
jobs:
  evaluate:
    foreach: ${{ needs.generate.outputs.items }}
    key: ${{ item.id }}
    uses: ./workflows/evaluate-one.yml
    with:
      item: ${{ item }}
```

Generated Jobごとに独立binding/Child Runを持つ。Nested Dynamicにも同じ規則。

Dynamic Job数1000制限はParent Run generated Job数へ適用。Child内generated JobはChild Run自身で数える。

## 18. Child concurrency

Childは自身のWorkflow concurrencyを持つ。Parent groupを自動継承しない。

Childがconcurrency waitになってもParent Jobは`waiting_child`を維持する。

## 19. Recovery

Runtime再起動後:

- Parent Attempt + binding + Child relation復元
- Child runningなら通常Recovery
- Child completedならParent未反映結果をidempotentに伝播
- same Parent AttemptへChild重複作成禁止
- Retry時もbindingをresolverで差し替えない

## 20. Persistence uniqueness

- one binding per `parent_job_run_id`
- one Child Run per `parent_attempt_id`

をDB unique constraintで保証する。

## 21. Events

```text
reusable_binding_created
child_workflow_started
child_workflow_completed
child_workflow_cancel_propagated
child_workflow_cycle_rejected
child_workflow_direct_control_rejected
```

## 22. Failure code

```text
workflow_not_found
workflow_reference_invalid
workflow_input_invalid
workflow_cycle_detected
child_workflow_start_failed
child_workflow_failed
child_workflow_cancelled
workflow_output_invalid
action_version_mismatch
child_run_direct_control_forbidden
```

## 23. 受入条件

1. file/registered ID reference
2. URL/absolute/path traversal拒否
3. binding first activation
4. binding後source変更でもRetry固定
5. parent-child success/failure/cancel
6. Workflow `jobs` output mapping
7. state isolation
8. direct/indirect cycle
9. Retry -> new Child / same binding
10. public Child control拒否/read許可
11. Parent Pauseはstarted Childへ非伝播
12. Dynamic + Reusable
13. runtime restart duplicate防止
14. one binding/job + one child/attempt uniqueness
