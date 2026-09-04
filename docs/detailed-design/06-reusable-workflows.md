# 06. Reusable Workflows 詳細設計

- Status: Draft v1.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `05`, `08`, `09`, `10`, `11`, `12`

## 1. 目的

Reusable Workflow の参照、親子Workflow Run、Input/Output/ArtifactRef、Definition binding、Action/Validator version、System baseline、effective settings、priority、Concurrency、Retry/Cancel/Recoveryを定義する。

## 2. 基本原則

1. 親から見るReusable Workflowは1 concrete Job。
2. 子は独立Workflow Run。
3. 親子mutable stateは共有しない。
4. Data flowはInput/Outputへ明示mapping。
5. Artifact実体は暗黙共有せずArtifactRefを明示的に渡す。
6. `wf_start`ではReusable graphを事前検証するがbindingは作らない。
7. Child DefinitionをParent concrete Jobの最初のactivation時にfresh readしてbindingへsnapshotする。
8. Child System workflow defaults / effective runtime settings / Retentionをbinding時にsnapshot。
9. Cycle禁止。固定depth limit無し。
10. Parent Job Retryは最初のChild bindingを再利用。
11. BindingはChild Action/Validator version固定。
12. Child Run priorityはroot Workflow Run priorityを継承する。
13. Child ConcurrencyはChild `workflow_id + group` scopeで通常Workflow Runと同じ規則を使う。

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

## 4. `uses` reference / Resolver source contract

`uses`=literal stringのみ。

MVPで**実行可能な全Workflow参照**はWorkflowResolverから以下を取得できることを必須とする。

```text
canonical workflow_id
UTF-8 YAML source bytes
source_kind
source_display
filesystem_base_dir optional
```

Registered Workflow IDも最終的には現在のYAML source bytesを返す。Typed objectだけを返してsource YAML bytesを提供できない登録方式はMVPの実行対象にしない。Source bytes unavailable=`workflow_source_unavailable`。

Relative reference基準=caller Workflow sourceの`filesystem_base_dir`。

- filesystem callerのみrelative可
- `.yml|.yaml`
- absolute path禁止
- `..`はcanonical resultがWorkflow root内なら可
- symlink解決後root外reject

`./`/`../`で始まらないnon-file literalはRegistered Workflow ID。

Non-filesystem callerはrelative不可。URL/HTTP/Git direct fetch無し。

## 5. Run-start Reusable preflight

Root `wf_start`では、その時点で解決できるcurrent sourceを使って**Reusable graph全体を再帰的に事前検証**する。

目的は明らかに実行不能なWorkflowをRun row作成前にrejectすることであり、Child Definitionのbinding固定ではない。

Preflight:

1. Root source snapshotから`uses` Jobを列挙
2. 各referenceをResolverでcurrent source bytesへ解決
3. source unavailable/path escape/reference invalidをreject
4. current Child bytesをstrict parse/validate
5. canonical workflow_idでrecursive ancestor cycle検出
6. Child内`uses`も同様に再帰検証
7. current Child Action/Validator IDがRegistryに存在することを検証
8. resolved internal Runner Poolが登録済みであることを検証
9. JSON Schema/Expression/static Definition constraintsを通常どおり検証

Preflight中に:

- `reusable_bindings` rowを作らない
- Child Workflow Runを作らない
- Child Definition/version/settingsをParent Run snapshotへbindingとして固定しない

したがってRoot Run start後、Reusable Job activation前にChild sourceが変更されることは許容される。**first binding時のfresh readが実際に使うChild DefinitionのSource of Truth**である。

Preflightでpassした後でもfirst binding時にsourceがinvalid/unavailable/version不整合になっていればParent activationをfail-closedする。逆にpreflight後の変更をold preflight結果で黙って通さない。

## 6. Binding creation / fresh Child source read

Parent concrete Job最初のactivation時にreferenceを解決し、`reusable_bindings`へ1回だけ固定する。

**新binding作成時はRun-start preflight結果やWorkflowResolver cacheだけを使わず、`01 §19.2`と同じexecution-time source readを必ずやり直す。**

1. reference resolve
2. current Child UTF-8 YAML source bytesを1 logical readで取得
3. そのbytesをparse/strict validate/hash
4. source YAML + typed Definition/hashを同じbytesから作る
5. canonical workflow_idでcurrent ancestor chain cycle check
6. current Action/Validator versions・Runner Poolを再検証
7. invalid/source unavailableならold cache/preflight結果へfallbackせずParent activation failure

Binding作成後はChild sourceを再読込しない。Parent Retry/new Child Runはbinding snapshotを使う。

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

### 6.1 Child System baseline / settings resolution

Current System configをbinding時に読み直さない。

```text
child_system_workflow_defaults_json
  = parent Workflow Run.system_workflow_defaults_json exact copy
```

Child effective settings=`Child Workflow settings > inherited System baseline > canonical default`。

Child effective Retention=`Child Workflow retention > inherited System baseline retention > unlimited`。

Root Run開始後にSystem configが変わっても同一Run lineageのChild/Nested Child挙動は変わらない。

### 6.2 Binding validation

1. reference/current source bytes resolve
2. canonical child_workflow_id
3. current ancestor cycle check
4. Child strict static validation
5. Child Action current versions resolve
6. Child Validator current versions resolve
7. inherited System baselineからChild effective settings/Retention resolve
8. Child Runner Pool availability検証
9. binding persist

## 7. Child Run作成 / Concurrency admission

Parent Attempt作成時にsame bindingからChild Run開始を試みる。

Child concurrency expressionはChild Input/envで評価し、scope=`(child_workflow_id, resolved group)`。

### 7.1 Concurrencyなし / slotあり

1 transactionでParent Attempt -> waiting_child、Child Run `status=running`、parent/child relation、static Job rows、Event/Execution Logを作る。

Childはresolver/System configを再読込せずbinding snapshotを使う。

### 7.2 `on-limit=queue`

Child Runを作成し:

```text
Child status=queued
Child wait_reason=concurrency
Child concurrency_queued_at=current time
Parent Job/Attempt=waiting_child
```

slot取得後にChild `status=running, wait_reason=NULL, concurrency_queued_at=NULL`へ変更して通常Scheduling開始。

### 7.3 `on-limit=reject`

slot取得不可なら:

- Child Runは作らない
- Parent Attempt=`failure`
- code=`child_workflow_start_failed`
- details.reason=`concurrency_limit_reached`
- retryable=false default

Parent Job Retry policyで明示Retry可能。Retry時same bindingからChild admissionだけ再試行する。

## 8. Child Run snapshot

Child Runへcopy:

- binding Definition
- binding Action/Validator versions
- binding System baseline
- binding effective settings/Retention
- Parent `source_identity`（存在時）
- Parent ActorContext/AccessScope
- current root Workflow Run priority

## 9. Priority

Child Definition top-level priorityはReusable invocation時のChild Run priorityには使わない。

```text
Child Run priority = current root Workflow Run priority
```

Root `wf_priority_update` はroot + 全non-terminal descendant Child Runへ同値を伝播。Future Childもupdated root priorityを継承。Child direct priority update禁止。

## 10. Parent/Child identity

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

## 11. Input / ArtifactRef / State / Actor

Parent `with`をChild Workflow Input Schemaでstrict検証し、そのJSON objectをReusable Parent Attempt persistent Inputとしても保存する。

Secret禁止。

ArtifactRefは通常Input値として渡す。Cross-run Managed Artifact materializeはChild persistent Job InputへArtifactRefが流れ、`09/12` Authorization/data availabilityを満たす場合のみ。

Child state独立。Parent state直接read/write無し。

ActorContext/AccessScopeは継承し権限拡大禁止。

## 12. Child Output -> Parent Job Output

Child top-level `outputs`をChild success確定直前に評価しJSON objectとして永続化する。

Child success後、Parent Reusable AttemptはそのChild Workflow Output objectをresultとして受け取る。

Parent Reusable Jobに`outputs.schema`がある場合:

1. Child Workflow Output object取得
2. Parent Job Draft2020-12 Schema検証
3. SecretGuard
4. Parent Attempt PayloadStore
5. Parent Job success

Schema failure=`output_validation_failed`。

Artifactを返したい場合はChild Workflow Output fieldへArtifactRefを明示。Child ArtifactをParent artifacts namespaceへ自動mirrorしない。

## 13. Conclusion / Progress propagation

| Child Run | Parent Job |
| --- | --- |
| success + Parent Schema pass | success |
| success + Parent Schema fail | failure (`output_validation_failed`) |
| failure | failure (`child_workflow_failed`) |
| cancelled | cancelled |

Concurrency rejectでChild未作成ならParent=`failure(child_workflow_start_failed)`。

Parent `continue-on-error`は通常規則。Reusable progress autoはcurrent Child Workflow fractionを使う。Concurrency queue中Childは0扱い。

## 14. Retry

Parent Retry:

- Parent Job pending Input固定
- same binding
- new Parent Attempt
- Child concurrency admissionを再実行
- admitted/queuedならnew Child Run
- binding Definition/System baseline/settings/Retention/versions固定
- root priorityはRetry開始時current root priority

Current Registryがbinding Action/Validator versionsとexact一致必須。Child sourceはRetry時に再読込しない。

## 15. Result Reuse identity

Reusable Parent executor identity:

```text
child_definition_hash
child_action_versions_json
child_validator_versions_json
child_system_workflow_defaults_json
child_effective_settings_json
child_retention_policy_json
```

Root priority/Concurrency wait timingはidentity外。Parent persistent Input ArtifactRefはInput digestへ含む。

## 16. Cancel / Pause

Parent cancel -> current Child cancel。

Parent Pauseはstarted/queued Childへ伝播しない。Parent Pause中new Child Run開始は禁止。

Child public direct pause/resume/cancel/retry/priority updateは禁止=`child_run_direct_control_forbidden`。

Read/info/input/output/log/event/artifact/stateはAuthorization範囲内で許可。

## 17. Cycle / depth

Canonical workflow_id ancestor chainでcycle検出。固定depth無し。call_depth保存。

Run-start preflightではcurrent graph全体のcycleを検出し、first bindingでもcurrent ancestor chainを再検証する。

## 18. Dynamic Job

Reusable Dynamic template可。Generated concrete Jobごとbinding/Child Run。

Parent generated上限=Parent Run、Child内=Child Run effective setting。

## 19. Child concurrency

Child Definition concurrencyをChild Input/envから評価。Parent group自動継承無し。Scope/Capacity/queue/reject=`01/08`。

Child concurrency wait中Parent Job=waiting_child。

## 20. Recovery / uniqueness

Restart:

- binding/relation/System baseline/settings復元
- binding Action/Validator version availability確認
- Child running/queued concurrency wait通常Recovery
- Child completed Parentへidempotent propagation
- root priority repair
- same Parent Attempt Child重複作成無し

DB:

- one binding / parent_job_run_id
- one Child / parent_attempt_id when Child created

Concurrency rejectでChild未作成のfailed Parent Attemptはrelation無しが正規状態。

## 21. Failure code

```text
workflow_not_found
workflow_source_unavailable
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

## 22. 受入条件

1. all executable refs provide UTF-8 YAML source bytes
2. relative/registered reference/path safety
3. wf_start recursively preflights current reusable graph but creates no binding/Child row
4. preflight detects current source invalid/missing/cycle/action-validator/pool errors
5. first binding always re-reads current Child source bytes independent of preflight/cache
6. source change between wf_start and first activation is allowed and binding-time source wins
7. invalid/unavailable/current-version mismatch at binding never falls back to preflight/cache
8. Retry does not reread Child source
9. Child System baseline inherits Parent Run snapshot
10. Child effective settings/Retention snapshot
11. nested Child same baseline
12. binding Definition+Action+Validator versions
13. admitted Child=running / queued only for concurrency wait + queue timestamp
14. Child priority/root update propagation
15. Child concurrency workflow_id scope
16. Child queue -> Child queued/Parent waiting
17. Child reject -> no Child row/Parent failure
18. Retry after reject same binding
19. Reusable Parent outputs.schema
20. Child state isolation
21. ArtifactRef both directions/no mirror
22. cross-run Artifact authorization
23. Parent progress
24. cycle/depth/direct Child control
25. Dynamic+Reusable/recovery duplicate prevention
