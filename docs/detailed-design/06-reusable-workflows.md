# 06. Reusable Workflows 詳細設計

- Status: Draft v1.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `05`, `08`, `09`, `10`, `11`, `12`

## 1. 目的

Reusable Workflow の参照、親子Workflow Run、Input/Output/ArtifactRef、Definition binding、Action/Validator version、System baseline、effective settings、priority、Retry/Cancel/Recoveryを定義する。

## 2. 基本原則

1. 親から見るReusable Workflowは1 concrete Job。
2. 子は独立Workflow Run。
3. 親子mutable stateは共有しない。
4. Data flowはInput/Outputへ明示mapping。
5. Artifact実体は暗黙共有せずArtifactRefを明示的に渡す。
6. Child Definitionをsnapshot。
7. Child System workflow defaults / effective runtime settings / Retentionをbinding時にsnapshot。
8. Cycle禁止。固定depth limit無し。
9. Parent Job Retryは最初のChild bindingを再利用。
10. BindingはChild Action/Validator version固定。
11. Child Run priorityはroot Workflow Run priorityを継承する。

## 3. YAML

```yaml
jobs:
  analyze:
    uses: ./common/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
    outputs:
      schema:
        type: object
```

ArtifactRefも通常`with`値として渡す。

`uses` Jobでは`action/validator/executor/runs-on/success_if/external/timeout-minutes`禁止。`outputs.schema`はoptional。

## 4. `uses` reference

`uses`=literal stringのみ。

Relative reference基準=caller Workflow source file directory。

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
child_system_workflow_defaults_json
child_effective_settings_json
child_retention_policy_json
created_at
```

### 5.1 Child System baseline / settings resolution

**Current System configをbinding時に読み直さない。**

Child System baseline:

```text
child_system_workflow_defaults_json
  = parent Workflow Run.system_workflow_defaults_json exact copy
```

Child effective settings:

```text
Child Workflow settings > inherited System baseline > canonical default
```

Child effective Retention:

```text
Child Workflow retention > inherited System baseline retention > unlimited
```

したがってRoot Run開始後にSystem configが変更されても、同一Run lineageの後続Reusable Child/Nested Childの挙動は変わらない。

Child `default_runner_pool`、Dynamic上限、Output threshold、External default lease、Progress/Log mode等もこのbaseline/effective snapshotから解決する。

Binding後はChild source/Registry/System config変更をParent Retryへ自動反映しない。

### 5.2 Binding validation

1. reference resolve
2. canonical child_workflow_id
3. ancestor cycle check
4. Child static validation
5. Child Action current versions resolve
6. Child Validator current versions resolve
7. inherited System baselineからChild effective settings/Retention resolve
8. Child default/explicit Runner Pool availability検証
9. binding persist

## 6. Child Run作成

Parent Attempt作成時、bindingからexactly one Child Run。

Atomic:

- Parent Attempt -> waiting_child
- Child Workflow Run
- parent/child relation
- Child static non-dynamic Job rows
- Event/Execution Log

ChildはresolverからDefinition/System configを再読込せずbinding snapshotを使う。

Child Runへcopy:

- binding Definition
- binding Action/Validator versions
- binding System baseline
- binding effective settings/Retention
- Parent `source_identity`（存在時）
- Parent ActorContext/AccessScope
- current root Workflow Run priority

## 7. Priority

Child Definition top-level `priority`はReusable invocation時のChild Run priorityには使わない。

```text
Child Run priority = current root Workflow Run priority
```

Root `wf_priority_update` はroot + 全non-terminal descendant Child Runへ同値を伝播する。Future Childも更新後root priorityを継承。

Child direct priority updateは禁止。

これによりReusable内部Jobもroot Workflow Runの運用priorityでSchedulingされる。Child Job `priority`は通常どおり二次軸として有効。

## 8. Parent/Child identity

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

## 9. Input / ArtifactRef / State / Actor

Parent `with`をChild Workflow Input Schemaで検証し、そのJSON objectをReusable Parent Attempt persistent Inputとして保存する。

Secret禁止。

ArtifactRefは通常Input値として渡す。専用mount/port無し。

Cross-run Managed Artifact materializeはChild persistent Job InputへArtifactRefが流れ、`09/12` Authorization/data availabilityを満たす場合のみ。

Child state独立。Parent state直接read/write無し。

ActorContext/AccessScopeは継承し権限拡大禁止。

## 10. Child Output -> Parent Job Output

Child top-level `outputs`をChild success確定直前に評価しJSON objectとして永続化する。

Child success後、Parent Reusable AttemptはそのChild Workflow Output objectをresultとして受け取る。

Parent Reusable Jobに`outputs.schema`がある場合:

1. Child Workflow Output object取得
2. Parent Job Draft2020-12 Schema検証
3. SecretGuard
4. Parent Attempt PayloadStoreへ保存
5. Parent Job success

Schema failure=`output_validation_failed`。Parent Attempt failureとなり通常Retry policy対象。

Artifactを返したい場合はChild Workflow Output fieldへArtifactRefを明示する。Child ArtifactをParent `needs.<job>.artifacts`へ自動mirrorしない。

## 11. Conclusion / Progress propagation

| Child Run | Parent Job |
| --- | --- |
| success + Parent Output Schema pass | success |
| success + Parent Output Schema fail | failure (`output_validation_failed`) |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Parent `continue-on-error`は通常規則。

Parent Reusable Job `progress.mode=auto` はcurrent Child Workflow progress fractionを利用。Child terminalでParent Job fraction=1。

## 12. Retry

Automatic/ManualいずれもParent Retryは:

- Parent Job pending Input snapshot固定
- same reusable binding
- new Parent Attempt
- new Child Workflow Run
- binding Definition/System baseline/settings/Retention/versions固定
- root priorityはRetry開始時current root priorityをChild Runへcopy

Retry開始前にcurrent Registryがbinding Action/Validator versionsとexact一致することを要求。

ArtifactRefを別generationへ差し替えない。

## 13. Result Reuse identity

Reusable Parent executor identityは:

```text
child_definition_hash
child_action_versions_json
child_validator_versions_json
child_system_workflow_defaults_json
child_effective_settings_json
child_retention_policy_json
```

を含む。

Root scheduling priorityはresult semanticsではないためreuse identityへ含めない。

Parent persistent Input ArtifactRefはInput digestへ含む。

## 14. Cancel / Pause

Parent cancel -> current Child cancel。

Parent Pauseはstarted Childへ伝播しない。Pause中new Child Run開始禁止。

Child public direct:

```text
pause/resume/cancel/retry/priority update
```

は禁止=`child_run_direct_control_forbidden`。

Read/info/input/output/log/event/artifactはAuthorization範囲内で許可。

## 15. Cycle / depth

Canonical workflow_id ancestor chainでcycle検出。固定depth無し。call_depth保存。

## 16. Dynamic Job

Reusable Dynamic template可。Generated concrete Jobごとbinding/Child Run。

Parent generated上限=Parent Run、Child内はChild binding effective setting。

## 17. Child concurrency

Child Definition concurrencyをChild Input/envから評価。Parent group自動継承無し。

Child concurrency wait中Parent Job=waiting_child。

## 18. Recovery / uniqueness

Restart:

- binding/relation/System baseline/settings復元
- binding Action/Validator version availability確認
- Child running通常Recovery
- Child completed Parentへidempotent propagation
- root priorityとの不一致があればnon-terminal Childをroot priorityへrepair
- same Parent Attempt Child重複作成無し

DB:

- one binding / parent_job_run_id
- one Child / parent_attempt_id

## 19. Failure code

```text
workflow_not_found
workflow_reference_invalid
workflow_relative_reference_unavailable
workflow_cycle_detected
workflow_input_invalid
child_workflow_start_failed
child_workflow_failed
workflow_output_invalid
output_validation_failed
action_version_mismatch
validator_version_mismatch
artifact_access_forbidden
artifact_data_unavailable
child_run_direct_control_forbidden
```

## 20. 受入条件

1. relative/registered reference/path safety
2. Child System baseline inherits Parent Run snapshot, not current System
3. Child effective settings/Retention snapshot
4. nested Child uses inherited baseline
5. System change after Root start does not alter later Child
6. binding Definition+Action+Validator versions
7. Child priority=root priority
8. root priority update propagates descendants
9. Retry same binding/version fail-closed
10. Reusable Parent outputs.schema pass/fail
11. Child state isolation
12. parent->child / child->parent ArtifactRef, no mirror
13. cross-run Artifact authorization
14. Parent auto progress mirrors Child
15. cycle/depth/direct Child control
16. Dynamic+Reusable/restart duplicate prevention
