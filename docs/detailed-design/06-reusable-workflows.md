# 06. Reusable Workflows 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `05`, `08`, `09`, `10`

## 1. 目的

Reusable Workflow の参照、親子Workflow Run、Input/Output/ArtifactRef、Definition binding、Action/Validator version、effective settings、Retry/Cancel/Recoveryを定義する。

## 2. 基本原則

1. 親から見るReusable Workflowは1 concrete Job。
2. 子は独立Workflow Run。
3. 親子mutable stateは共有しない。
4. Data flowはInput/Outputへ明示mapping。
5. Artifact実体は暗黙共有せずArtifactRefを明示的に渡す。
6. Child Definitionをsnapshot。
7. Child effective runtime settings/Retentionもbinding時にsnapshot。
8. Cycle禁止。固定depth limit無し。
9. Parent Job Retryは最初のChild bindingを再利用。
10. BindingはChild Action/Validator version固定。

## 3. YAML

```yaml
jobs:
  analyze:
    uses: ./common/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
```

ArtifactRef:

```yaml
jobs:
  analyze:
    needs: [export]
    uses: ./common/analyze.yml
    with:
      source_artifact: ${{ needs.export.artifacts.dataset }}
```

`uses` Jobでは`action/validator/executor/runs-on/success_if/external/timeout-minutes`禁止。Executor resolutionは`01`。

## 4. `uses` reference

`uses`=literal stringのみ。

Relative:

```yaml
uses: ./common/analyze.yml
```

基準=caller Workflow source file directory。

- filesystem callerのみrelative可
- `.yml|.yaml`
- absolute path禁止
- `..`はcanonical resultがWorkflow root内なら可
- symlink解決後root外reject

`./`/`../`で始まらないnon-file literalはRegistered Workflow ID。

Non-filesystem callerはrelative不可。URL/HTTP/Git direct fetch無し。

## 5. Binding

Parent concrete Job最初のactivation時にreferenceを解決し、`reusable_bindings`へ1回だけ固定。

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
child_effective_settings_json
child_retention_policy_json
created_at
```

### 5.1 Child settings resolution

Binding作成時に:

```text
Child Workflow settings > current Parent Runtime System config > canonical default
```

でChild effective settings/Retentionを計算する。

このbinding作成後はSystem config/Child source settingsが変わっても、同じParent Job RunのRetryでは**binding snapshotを再利用**する。

Child `default_runner_pool`、Dynamic上限、Output threshold、External default lease、Progress/Log mode等もこのsnapshotから解決する。

Child retentionはChild Run自身にsnapshotする。ただしChildはParent Run所有subtreeなので、Parent run-history retentionでParent subtreeが削除される場合はParentの寿命がChildの実効上限になる。

### 5.2 Binding validation

1. reference resolve
2. canonical child_workflow_id
3. ancestor cycle check
4. Child static validation
5. Child Action current versions resolve
6. Child Validator current versions resolve
7. Child effective settings/Retention resolve
8. binding persist

Binding後Child source/Registry/System settings変更をParent Retryへ自動反映しない。

## 6. Child Run作成

Parent Attempt作成時、bindingからexactly one Child Run。

Atomic:

- Parent Attempt -> waiting_child
- Child Workflow Run
- parent/child relation
- Child static non-dynamic Job rows
- Event/Execution Log

ChildはresolverからDefinitionを再読込せずbinding snapshotを使う。

Child Runへ:

- binding Definition
- binding Action/Validator versions
- binding effective settings/Retention
- Parent Run `source_identity`（存在時）
- Parent ActorContext/AccessScope

をsnapshotする。

## 7. Parent/Child identity

Child:

```text
parent_workflow_run_id
parent_job_run_id
parent_attempt_id
root_workflow_run_id
call_depth
reusable_binding_id
```

Root=parent null, root=self, depth0。

## 8. Input / ArtifactRef / State / Actor

Parent `with`をChild Workflow Input Schemaで検証し、そのJSON objectをReusable Parent Attempt persistent Inputとしても保存する。

Secret禁止。

ArtifactRefは通常Input値として渡す。専用mount/port無し。

Cross-run Managed Artifact materializeはChild persistent Job InputへArtifactRefが流れ、`09/12` Authorization/data availabilityを満たす場合のみ。

Child state独立。Parent state直接read/write無し。

ActorContext/AccessScopeは継承し権限拡大禁止。

## 9. Child Output -> Parent Job Output

Child top-level `outputs` success直前評価。JSON objectをParent Reusable Job Outputにする。

Artifactを返す場合はChild Workflow Output fieldへArtifactRefを明示。

```yaml
outputs:
  report_artifact: ${{ jobs.report.artifacts.report }}
```

Parentは:

```text
needs.analyze.outputs.report_artifact
```

で参照。

Child ArtifactをParent `needs.analyze.artifacts`へ自動mirrorしない。

PayloadStoreはChild binding effective thresholdを使用。

## 10. Conclusion / Progress propagation

| Child Run | Parent Job |
| --- | --- |
| success | success |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Parent `continue-on-error`は通常規則。

Parent Reusable Job `progress.mode=auto` の間は`09`どおりcurrent Child Workflow progress fractionをParent Job fractionとして利用する。Child terminalでParent Job fraction=1。

## 11. Retry

Parent Retry:

- Parent Job pending Input snapshot固定
- new Parent Attempt
- same reusable binding
- new Child Workflow Run
- binding Definition/settings/Retention/versions固定

Retry開始前にcurrent Registryがbinding Action/Validator versionsとexact一致することを要求。

不足=`action_version_mismatch|validator_version_mismatch`, retryable=false。

ArtifactRefを別generationへ差し替えない。

## 12. Result Reuse identity

Reusable Parent executor identity:

```text
child_definition_hash
child_action_versions_json
child_validator_versions_json
child_effective_settings_json
child_retention_policy_json
```

を含む。

Parent persistent Input ArtifactRefはInput digestへ含む。

## 13. Cancel / Pause

Parent cancel -> current Child cancel。

Parent Pauseはstarted Childへ伝播しない。Pause中new Child Run開始禁止。

Child public direct:

```text
pause/resume/cancel/retry/priority update
```

は禁止=`child_run_direct_control_forbidden`。

Read/info/output/log/event/artifactはAuthorization範囲内で許可。

## 14. Cycle / depth

Canonical workflow_id ancestor chainでcycle検出。固定depth無し。call_depth保存。

## 15. Dynamic Job

Reusable Dynamic template可。Generated concrete Jobごとbinding/Child Run。

Parent generated上限=Parent Run、Child内はChild Run自身のbinding effective setting。

## 16. Child concurrency

ChildはChild Definition concurrencyを自身のInput/envから評価。Parent group自動継承無し。

Child concurrency wait中Parent Job=waiting_child。

## 17. Recovery / uniqueness

Restart:

- binding/relation/settings復元
- binding Action/Validator version availability確認
- Child running通常Recovery
- Child completed Parentへidempotent propagation
- same Parent Attempt Child重複作成無し

DB:

- one binding / parent_job_run_id
- one Child / parent_attempt_id

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
artifact_access_forbidden
artifact_data_unavailable
child_run_direct_control_forbidden
```

## 19. 受入条件

1. relative/registered reference/path safety
2. binding Definition+Action+Validator versions
3. binding Child effective settings/Retention
4. System setting changes after binding do not alter Retry Child
5. Child source_identity/Actor/Scope inheritance
6. Retry same binding/version fail-closed
7. Child state isolation
8. parent->child ArtifactRef
9. child->parent ArtifactRef/no mirror
10. cross-run Artifact authorization
11. Parent auto progress mirrors Child
12. cycle/depth
13. direct Child control reject
14. Dynamic+Reusable
15. restart duplicate prevention
