# 05. Dynamic Jobs 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`, `03-runtime-and-scheduling.md`

## 1. 目的

`foreach`によるDynamic Jobの生成、任意深さの入れ子、stable key、full logical Job key、atomic expansion、ordering、dependency group、Recoveryを正規化する。

## 2. 基本原則

1. Actionから任意Jobを直接追加せずYAML templateをEngineが展開する。
2. 生成Jobは通常Jobと同じJob Run/Attempt/Retry/Log/Artifactを持つ。
3. 入れ子深さに固定上限を置かない。
4. Workflow Run全体の**生成Job数**で上限管理し既定1000。
5. 1 expansionはall-or-nothing。
6. stable keyを優先し、識別にはparent pathを含むfull logical `job_key`を使う。
7. 同一Workflow Run internal Job同時実行最大1はDynamic Jobにも適用する。

## 3. Root Dynamic Job

```yaml
jobs:
  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    action: evaluate_candidate
    with:
      candidate: ${{ item }}
```

`evaluate`自体はActionを実行しないtemplate。配列要素ごとにgenerated Jobを作る。

## 4. Nested Dynamic Job の正規syntax

Nested expansionは`foreach` object formを使う。

```yaml
jobs:
  condition:
    foreach:
      parent: evaluate
      items: ${{ iteration.parent.outputs.conditions }}
    key: ${{ item.id }}
    action: evaluate_condition
    with:
      candidate: ${{ iteration.parent.item }}
      condition: ${{ item }}
```

規則:

- `parent`: 同じWorkflow内のDynamic Job template ID。literal stringのみ。
- `items`: parent generated Jobごとに評価するCEL expression。
- `parent`は明示的なDynamic parent edgeであり、同じtemplate IDを`needs`へ重複記載しない。
- `needs`にはparent以外のglobal dependencyを追加できる。global dependencyは各nested expansion前にterminal/condition要件を満たす必要がある。
- 1つのparent generated Jobにつき0または1 expansion instanceを作る。
- parent generated Jobがeffective successでない場合、defaultではそのparentに対するnested expansionを作らない。nested templateに明示`if: ${{ always() }}`等がある場合でも、Workflow cancel後は新規展開しない。

Root formの`foreach: <expr>`とNested object form以外はrejectする。

## 5. `foreach` result

`items`/root `foreach`はJSON array必須。`null/object/scalar`は`dynamic_foreach_type_error`。

空arrayは正常:

- generated count 0
- expansion complete
- dependency groupは空集合としてterminal
- 後続を永久待機させない

Source arrayをexpansion時にsnapshotする。

## 6. `item` / iteration snapshot

各generated Jobは`02`のexact shapeを保存する。

`iteration.current`:

```text
template_id
key
item
job_key
source_order
```

Nestedでは`iteration.parent`に直近parent generated Jobの:

```text
template_id/key/item/job_key/source_order/status/conclusion/outputs/artifacts
```

をsnapshotする。`iteration.ancestors`はoutermost -> direct parent順。

Retryではitem/iterationを再評価しない。

## 7. Stable raw key

`key`指定時の結果はstringまたはinteger。integerはbase-10 stringへcanonical conversion。空stringは禁止。

制約:

- raw key UTF-8 <= 256 bytes
- duplicate raw keyは**同一 expansion instance内**でfailure

`key`省略はsource indexをraw keyとして使用可能。index fallbackはsource order変更で識別も変わるためvalidation warningを出せる。

## 8. Full logical `job_key`

親の異なるgenerated Job同士が衝突しないよう、full pathをcanonical identityとする。

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

### 8.1 key encoding

Path component内のraw keyはUTF-8 bytesをpercent-encodeする。

safe characters:

```text
A-Z a-z 0-9 . _ -
```

その他は `%HH` uppercase hex。`%`, `[`, `]`, `/` は必ずencodeする。

Template IDは`01`の静的Job ID制約によりpath delimiterを含まない。

Full `job_key` UTF-8 <= 2048 bytes。超過は`dynamic_job_key_too_long`。

DB primary IDはopaque `job_run_id`を別に持つ。

## 9. 生成数上限

System default:

```text
max_dynamic_jobs_per_workflow_run = 1000
```

Workflow `settings.max-dynamic-jobs`でoverride可能。

数える対象は生成されたJob Run総数。static Job/template metadata/Child Workflow内のgenerated Jobは**各Child Workflow Run自身の上限**で数える。

新expansionを加えるとlimit超過ならそのexpansionは0件登録しfailure。truncateしない。

## 10. Expansion identity

1 expansion instance:

```text
workflow_run_id
template_id
parent_generated_job_run_id nullable
```

にunique。

Rootはparent=null。Nestedはparent generated Jobごとに別instance。

`dynamic_expansions` rowへ:

```text
id
workflow_run_id
template_id
parent_job_run_id nullable
source_snapshot_json
source_digest
generated_count
status
failure_json nullable
created_at/completed_at
```

を保存する。Runtime crash後もcomplete expansionを再生成しない。

## 11. Atomic expansion

事前にmemory上で全candidateを構築し検証:

- source array JSON-compatible
- key type/length/duplicate
- full job_key collision/length
- Input/if/order expression評価可能性
- Action/version / Runner Pool / executor
- Retry/timeout
- total generated count limit
- parent/ancestor整合

その後1 SQLite transactionで:

1. expansion row create/update
2. all generated Job rows insert
3. item/iteration/order snapshot
4. expansion status complete
5. Event

1件でも失敗ならrollbackしgenerated Jobを0件に保つ。Failure後のexpansion rowを残す場合は、rollback後に別transactionでfailure記録してよいがgenerated Jobは作らない。

## 12. `if`

Root template `if`はroot expansion前に1回評価。

Nested template `if`はparent generated Jobごとにiteration context付きで評価。

false:

- そのexpansion instanceは0件の正常skipped expansionとして記録
- generated Jobを作らない

Expression errorはexpansion failure。

## 13. `order_by`

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
  - expr: ${{ item.id }}
    direction: asc
```

または:

```yaml
order_by: source_order
```

各criterionの型/混在禁止は`02`に従う。Expansion時にsort key/rankをsnapshotし再計算しない。

Runner選択順:

1. Workflow priority
2. Job priority
3. order rank
4. source order
5. ready_at
6. job_run_id

## 14. Template group `needs`

後段で:

```yaml
aggregate:
  needs: [evaluate]
```

とした場合、`evaluate` templateから**Workflow Run内に生成された全Root generated Job**をgroupとして扱う。

Nested template `condition`をneedsに指定した場合、そのtemplateの全parent expansionから生成された全Jobをgroupとして扱う。

Groupは全該当expansionがactivation可能性を失い、全generated Jobがterminalになった時点でterminal。

### 14.1 group conclusion

- 0件: `success`
- 全Jobがeffective success/skipped: `success`
- non-allowed failureあり: `failure`
- required blockedあり: `blocked`
- cancel由来あり: `cancelled`

実際の各Job conclusionは`needs.<template>.jobs`で保持する。

## 15. Output / Artifact aggregation

`02`の正規shape:

```text
needs.evaluate.jobs[full_job_key]
needs.evaluate.outputs[full_job_key]
needs.evaluate.artifacts[full_job_key]
```

例:

```yaml
with:
  result: ${{ needs.evaluate.jobs["evaluate[candidate_a]"].outputs }}
```

Nested:

```yaml
with:
  result: ${{ needs.condition.jobs["evaluate[candidate_a]/condition[x]"].outputs }}
```

Map keyはfull logical `job_key`。raw keyだけではない。

## 16. Retry

Generated Job Retryは同じJob Runに新Attemptを追加する。

固定:

- full job_key
- raw key
- item
- iteration
- input
- order rank

Source `foreach`を再評価しない。

## 17. Runtime Recovery

- committed expansion: そのまま利用
- transaction未commit: generated Job無しとして再評価可能
- generated success: success維持
- running: Runner recovery
- queued: queued維持
- expansion二重生成禁止

Recoveryだけで既存success generated Jobを再実行しない。

## 18. Reusable Workflow / External / Human

Dynamic templateに通常Jobと同様に以下を置ける。

- `uses` -> generated JobごとにChild Workflow Run
- `executor: external_llm` -> generated Jobごとにexternal task
- `executor: human` -> generated Jobごとにreview

field constraintは`01`。生成数上限はJob数に適用し、Child内Job数はChild Run側で別管理。

## 19. Cancel / Pause

Pause中は新expansionを開始しない。既にcommit済みgenerated Jobは保持。

Cancel後は新expansion禁止。未実行generated Jobは通常Cancel規則へ。

## 20. Failure code

```text
dynamic_foreach_type_error
dynamic_key_invalid
dynamic_duplicate_key
dynamic_job_key_too_long
dynamic_job_collision
dynamic_limit_exceeded
dynamic_order_type_error
dynamic_parent_invalid
dynamic_expansion_validation_failed
dynamic_expansion_storage_failed
```

## 21. 受入条件

1. Root 0/1/N件
2. stable key/index fallback
3. special chars percent encoding
4. duplicate key
5. parent別同raw keyが衝突しない
6. 2048 byte key limit
7. Nested `foreach.parent/items`
8. 3段以上nested / arbitrary-depth representative
9. iteration exact snapshot
10. 1000 allowed /1001 rollback
11. atomic expansion crash/restart
12. order_by型/順序
13. group status/conclusion
14. full key output/artifact lookup
15. Retry snapshot固定
16. Dynamic + Reusable/External/Human
17. Pause/Cancel no-new-expansion
