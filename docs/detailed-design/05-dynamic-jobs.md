# 05. Dynamic Jobs 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`

## 1. 目的

`foreach` によるDynamic Job生成、任意深さの入れ子、stable key、full logical `job_key`、atomic expansion、ordering、dependency group、Action/Validator binding、Recoveryを定義する。

## 2. 基本原則

1. Actionから任意Jobを直接追加しない。YAML templateをEngineが展開する。
2. generated Jobは通常Jobと同じJob Run/Attempt/Retry/Log/Artifact/Validator規則を持つ。
3. 入れ子深さに固定上限無し。
4. Workflow Run generated Job数上限default1000。
5. 1 expansion all-or-nothing。
6. identityはparent path込みfull `job_key`。
7. 同一Run internal Job同時最大1。

## 3. Root Dynamic Job

```yaml
jobs:
  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    action: evaluate_candidate
    validator: domain.validate_candidate
```

Template自身はAction/Validatorを実行せずgenerated Job群の論理親。

## 4. Nested Dynamic Job

```yaml
jobs:
  condition:
    foreach:
      parent: evaluate
      items: ${{ iteration.parent.outputs.conditions }}
    key: ${{ item.id }}
    action: evaluate_condition
```

- parent: same Workflow Dynamic template ID literal
- parentはDAG edge/cycle検証対象
- parentをneedsへ重複記載禁止
- needsにはparent以外のglobal dependency可
- condition dependency set=`{parent} ∪ needs`
- 1 parent generated Jobにつき0/1 expansion
- dependencies terminal後if評価
- cancel後alwaysでもnew expansion無し

Root scalar/Nested object form以外reject。

## 5. `foreach` result

JSON array必須。null/object/scalar -> `dynamic_foreach_type_error`。

空array:

- generated count0
- expansion terminal
- group completed/success

source arrayはsnapshot。

## 6. `item` / `iteration`

`iteration.current`:

```text
template_id/key/item/job_key/source_order
```

Nested parent:

```text
template_id/key/item/job_key/source_order
status/conclusion/continue_on_error
outputs/artifacts
```

ancestors outermost -> parent。Retryで再評価無し。

## 7. raw key / full job key

key result string|integer。integerはbase10 string。empty string禁止。

Duplicate raw keyはsame expansion instance内failure。key無しはsource index fallback。

Full path例:

```text
evaluate[candidate_a]
evaluate[candidate_a]/condition[x]
```

raw keyはUTF-8 percent-encode。safe=`A-Z a-z 0-9 . _ -`、他はuppercase `%HH`。

Full logical `job_key` fixed byte limit無し。DB TEXT、filesystem pathはopaque ID。

## 8. 生成数上限

System default1000。Workflow `settings.max-dynamic-jobs` override。

Countは当該Workflow Run generated Job。ChildはChild Runで別count。

超過時expansion全体0件登録でfailure。truncate禁止。

## 9. Expansion identity

```text
workflow_run_id
template_id
parent_generated_job_run_id nullable
```

Root parent null、Nestedはparent generated Jobごと。

## 10. Atomic expansion preflight

全candidateをmemory上で構築し、**DB insert前に全件**検証:

- foreach result
- raw key type/duplicate
- full job_key collision
- parent/DAG
- Input/if/order expression
- executor field constraints
- Runner Pool
- Action ID/version existence
- Validator ID/version existence（指定時）
- Reusable reference/binding prerequisites
- Retry policy/internal timeout
- generated count limit

Action/Validator versionはWorkflow Run snapshotと一致すること。Expansion中にRegistryが変わって不一致なら `dynamic_expansion_validation_failed` で全rollback。

1 SQLite transactionでgenerated Job、Action/Validator version、item/iteration/order snapshot、expansion metadataを保存。

1件でも失敗ならgenerated Jobを残さない。

## 11. `if` / order

Root: declared needs terminal後1回。
Nested: parent+needs terminal後parentごと。

false -> 0件skipped expansion。Expression error -> expansion failure。

`order_by`は`02`のfinite string/number規則。Sort rank snapshot。

Runner選択:

1. Workflow priority
2. Job priority
3. Dynamic rank
4. source order
5. ready_at
6. job_run_id

## 12. Template group

Template `needs` は当該Run generated Job群全体。

Status:

- 未確定/non-terminalあり: running
- all expansion known + all generated terminal: completed

Conclusion:

- 0件: success
- all effective success/skipped: success
- non-allowed failure: failure
- required blocked: blocked
- cancel: cancelled

## 13. Output / Artifact aggregation

```text
needs.<template>.jobs[full_job_key]
needs.<template>.outputs[full_job_key]
needs.<template>.artifacts[full_job_key]
```

raw keyだけでindexしない。

## 14. Retry / Recovery

Generated Job Retryはsame Job Run new Attempt。

固定:

- full/raw key
- item/iteration
- persistent Input
- order rank
- Action/Validator version

foreach再評価無し。

Recovery:

- committed expansion再生成無し
- uncommittedは0件として再評価
- success保持
- running Runner recovery
- queued維持

## 15. Reusable / External / Human

Dynamic templateにReusable/External/Human可。Field/Validator constraintsは`01`。

External generated Jobもoptional Validator可。Human/Reusable generated JobはValidator禁止。

## 16. Pause / Cancel

Pause中new expansion無し。commit済み保持。

Cancel後new expansion無し。未実行generatedは通常cancel。

## 17. Failure code

```text
dynamic_foreach_type_error
dynamic_key_invalid
dynamic_duplicate_key
dynamic_job_collision
dynamic_limit_exceeded
dynamic_order_type_error
dynamic_parent_invalid
dynamic_cycle_detected
dynamic_expansion_validation_failed
dynamic_expansion_storage_failed
```

## 18. 受入条件

1. Root/Nested/3+ depth
2. parent cycle
3. parent+needs helper
4. parent別same raw key
5. no fixed job_key limit
6. 1000/1001 rollback
7. Action+Validator version preflight before insert
8. unknown/mismatched Validator全rollback
9. atomic recovery
10. group status/conclusion
11. full-key output/artifact
12. Retry Action+Validator snapshot固定
