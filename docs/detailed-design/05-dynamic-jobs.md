# 05. Dynamic Jobs 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`

## 1. 目的

`foreach` による Dynamic Job の生成、任意深さの入れ子、stable key、full logical `job_key`、atomic expansion、ordering、dependency group、Recovery を定義する。

## 2. 基本原則

1. Actionから任意Jobを直接追加しない。YAML templateをEngineが展開する。
2. generated Job は通常Jobと同じ Job Run / Attempt / Retry / Log / Artifact を持つ。
3. 入れ子深さに固定上限を置かない。
4. Workflow Run全体の generated Job数上限で暴走防止し、既定1000。
5. 1 expansionはall-or-nothing。
6. logical identity は親pathを含む full `job_key`。
7. 同一Workflow Run internal Job同時実行最大1はgenerated Jobにも適用。

## 3. Root Dynamic Job

```yaml
jobs:
  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    action: evaluate_candidate
```

Template自身はActionを実行せず、generated Job群の論理親となる。

## 4. Nested Dynamic Job

正規syntax:

```yaml
jobs:
  condition:
    foreach:
      parent: evaluate
      items: ${{ iteration.parent.outputs.conditions }}
    key: ${{ item.id }}
    action: evaluate_condition
```

規則:

- `parent`: 同一Workflow内のDynamic template ID、literal string。
- `parent`は依存edgeでありDAG cycle検証対象。
- 同じparentを`needs`へ重複記載してはならない。
- `needs`にはparent以外のglobal dependencyを追加可能。
- condition dependency set は `{parent} ∪ needs`。
- 1 parent generated Jobにつき0または1 expansion instance。
- parent/needsがterminal後に`if`を評価する。
- Workflow cancel後は`always()`でも新規expansionしない。

Root scalar formとNested object form以外はreject。

## 5. `foreach` result

JSON array必須。`null/object/scalar`は `dynamic_foreach_type_error`。

空配列は正常:

- generated count 0
- expansion terminal
- groupは `completed/success`
- 後続を永久待機させない

source arrayはexpansion時にsnapshot。

## 6. `item` / `iteration`

`02`のshapeを保存する。

`iteration.current`:

```text
template_id
key
item
job_key
source_order
```

Nested `iteration.parent`:

```text
template_id
key
item
job_key
source_order
status
conclusion
continue_on_error
outputs
artifacts
```

`iteration.ancestors`は outermost -> direct parent。

Retryでは再評価しない。

## 7. raw key

`key`結果は string または integer。integerはbase-10 string化。空string禁止。

同一 expansion instance内duplicate raw keyはfailure。

`key`省略時はsource index fallbackを許可する。

## 8. Full logical `job_key`

Root:

```text
evaluate[candidate_a]
```

Nested:

```text
evaluate[candidate_a]/condition[x]
evaluate[candidate_b]/condition[x]
```

さらに深い場合:

```text
evaluate[candidate_a]/condition[x]/scenario[1]
```

### 8.1 component encoding

raw keyはUTF-8 percent-encodeする。

safe:

```text
A-Z a-z 0-9 . _ -
```

`%`, `[`, `]`, `/` 等は `%HH` uppercase hex。

### 8.2 深さと長さ

**MVPでは full logical `job_key` に固定byte長上限を設けない。**

入れ子段数を任意にする決定と衝突しないためである。DBでは `TEXT` と opaque `job_run_id` を使用し、filesystem pathには `job_key` を直接使わない。

DoS/表示上の問題は generated Job数上限1000とAPI paginationで制御する。将来長さ上限を追加する場合は明示的なbreaking validation ruleとして扱う。

## 9. 生成数上限

System default:

```text
max_dynamic_jobs_per_workflow_run = 1000
```

Workflow `settings.max-dynamic-jobs` でoverride。

数えるのは当該Workflow Runに生成されたJob Run。Child Workflow内はChild Run自身で別カウント。

上限超過時はexpansion全体を0件登録でfailure。truncate禁止。

## 10. Expansion identity

unique identity:

```text
workflow_run_id
template_id
parent_generated_job_run_id nullable
```

Rootはparent null。Nestedはparent generated Jobごと。

`dynamic_expansions`にsource snapshot/digest/generated_count/status/failureを保存する。

## 11. Atomic expansion

memory上で全candidateを構築・検証:

- foreach結果
- raw key type/duplicate
- full job_key collision
- parent/DAG整合
- Input/if/order expression
- Action/version/Runner Pool/executor
- Retry/internal timeout
- generated count limit

1 SQLite transactionで全generated Jobとexpansion metadataを保存する。

1件でも失敗ならrollbackし generated Jobを残さない。

## 12. `if`

Root template: declared needs terminal後に1回評価。

Nested template: parent generated Job + declared needs terminal後、iteration付きでparentごとに評価。

helper意味は`02`。

falseはそのexpansion instanceを0件 `skipped` として確定。expression errorはexpansion failure。

## 13. `order_by`

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
```

または `source_order`。

型規則は`02`。sort key/rankはexpansion時snapshot。

Runner選択:

1. Workflow priority
2. Job priority
3. Dynamic order rank
4. source order
5. ready_at
6. job_run_id

## 14. Template group

`needs: [evaluate]` は `evaluate` templateから当該Runに生成された全該当Jobをgroupとして指す。

Nested templateは全parent expansionから生成された全Job。

### 14.1 status

- activation/expansionが未確定、またはgenerated Jobにnon-terminalあり: `running`
- 全expansion確定 + generated Job全terminal: `completed`

0件groupもexpansion確定後 `completed`。

### 14.2 conclusion

terminal時:

- 0件: `success`
- 全Job effective success/skipped: `success`
- non-allowed failureあり: `failure`
- required blockedあり: `blocked`
- cancel由来あり: `cancelled`

個別Jobの実conclusionはgroup mapに保持。

## 15. Output / Artifact aggregation

```text
needs.evaluate.jobs[full_job_key]
needs.evaluate.outputs[full_job_key]
needs.evaluate.artifacts[full_job_key]
```

raw keyのみでindexしない。

## 16. Retry / Recovery

Generated Job Retryは同じJob Runにnew Attempt。

固定:

- full job_key
- raw key
- item/iteration
- input
- order rank

foreachは再評価しない。

Recovery:

- committed expansionは再生成しない
- uncommittedは0件として再評価可能
- success generated Jobは保持
- runningはRunner recovery
- queuedは維持

## 17. Reusable / External / Human

Dynamic templateに `uses`, `executor: external_llm`, `executor: human` を置ける。field制約は`01`。

## 18. Pause / Cancel

Pause中は新expansion開始禁止。commit済みJobは保持。

Cancel後は新expansion禁止。未実行generated Jobは通常Cancel規則。

## 19. Failure code

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

## 20. 受入条件

1. Root/Nested/3段以上
2. parent edge cycle検出
3. parent+needs helper評価
4. parent別同raw key非衝突
5. fixed job_key length limit無し
6. 1000許可/1001 rollback
7. atomic expansion recovery
8. group status/conclusion
9. full key output/artifact lookup
10. Retry snapshot固定
