# 05. Dynamic Jobs 詳細設計

- Status: Draft v0.6
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
8. `foreach=[]` の正常0件と `if=false` のskipを区別する。
9. Nested parentが0件の場合はparent group conclusionを失わず伝播する。

## 3. Root Dynamic Job

```yaml
jobs:
  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    order_by:
      - expr: ${{ item.priority }}
        direction: desc
      - expr: ${{ item.id }}
        direction: asc
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
- 1 parent generated Jobにつき0/1 expansion instance
- dependencies terminal後if評価
- cancel後alwaysでもnew expansion無し

Root scalar/Nested object form以外reject。

## 5. Expansion outcome

各Root/Nested expansion instanceはJob statusとは別に:

```text
pending|expanded|skipped|failed|cancelled
```

を持つ。

- `if=false` -> `skipped`
- `foreach`評価成功かつarray（empty含む）をcommit -> `expanded`
- expression/preflight/storage error -> `failed`
- Workflow cancelで未展開 -> `cancelled`

`skipped` と empty `expanded` を同一扱いしない。

## 6. `foreach` result

JSON array必須。null/object/scalar -> `dynamic_foreach_type_error`。

空array:

- expansion=`expanded`
- generated count=0
- 正常な「仕事なし」
- Rootならtemplate groupはsuccess

source arrayはexpansion時snapshot。

## 7. Nested zero-parent propagation

Nested templateのparent groupがterminalになった時点でgenerated parent Jobが0件なら、per-parent expansion instanceは作れないため `iteration.parent` を使った`if/foreach`を評価しない。

代わりにparent group conclusionを基準にNested template groupを直接terminal化する。

| parent group | nested group (generated parent=0) |
| --- | --- |
| `success` | `success` |
| `skipped` | `skipped` |
| `failure` | `blocked` |
| `blocked` | `blocked` |
| `cancelled` | `cancelled` |

したがって:

- parent `foreach=[]` -> parent success -> nested success
- parent Root `if=false` -> parent skipped -> nested skipped
- parent expansion failure before any generated Job -> nested blocked

このzero-parent propagationは`if: always()`等で「存在しないparent item」を人工的に作る機能ではない。Parent itemが0件ならNested Actionは0件のまま。

Declared global `needs` にfailure/cancelがある場合は通常dependency規則を合わせて、より強い `cancelled > blocked/failure > skipped > success` の方向へterminal化する。

## 8. `item` / `iteration`

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

## 9. raw key / full job key

key result string|integer。integerはbase10 string。empty string禁止。

Duplicate raw keyはsame expansion instance内failure。key無しはsource index fallback。

Full path例:

```text
evaluate[candidate_a]
evaluate[candidate_a]/condition[x]
```

raw keyはUTF-8 percent-encode。safe=`A-Z a-z 0-9 . _ -`、他はuppercase `%HH`。

Full logical `job_key` fixed byte limit無し。DB TEXT、filesystem pathはopaque ID。

## 10. 生成数上限

System default1000。Workflow `settings.max-dynamic-jobs` override。

Countは当該Workflow Run generated Job。ChildはChild Runで別count。

超過時expansion全体0件登録でfailure。truncate禁止。

## 11. Expansion identity

```text
workflow_run_id
template_id
parent_generated_job_run_id nullable
```

Root parent null、Nestedはparent generated Jobごと。

## 12. Atomic expansion preflight

全candidateをmemory上で構築し、DB insert前に全件検証:

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

Action/Validator versionはWorkflow Run snapshotと一致。Expansion中Registry不一致なら `dynamic_expansion_validation_failed` で全rollback。

1 SQLite transactionでgenerated Job、Action/Validator version、item/iteration/order snapshot、expansion metadataを保存。

1件でも失敗ならgenerated Jobを残さない。

## 13. `if`

Root: declared needs terminal後1回。
Nested: parent generated Job + declared needs terminal後parentごと。

- true -> foreach評価へ
- false -> そのexpansion instance=`skipped`、foreachは評価しない、generated Job 0
- expression error -> expansion=`failed`

Nested parent generated Jobが0件の場合は§7を使い、per-parent `if`は評価しない。

Workflow cancel後は新規if/foreach activationを開始しない。

## 14. `order_by` canonical schema

未指定時はsource array order。

明示source order:

```yaml
order_by: source_order
```

Expression sort:

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
  - expr: ${{ item.id }}
    direction: asc
```

許可形:

```text
null/omitted
literal "source_order"
non-empty array of {expr, direction}
```

`direction=asc|desc`、default `asc`。Unknown key reject。

各criterionは全candidateで同一型のstringまたはfinite number。null/bool/object/array/NaN/Infinity/criterion内混在型は`dynamic_order_type_error`。

Sortはcriterionを左からlexicographic適用し、全criterion同値ならsource_order ASCでstable tie-break。最終整数`order_rank`をexpansion内0..N-1でsnapshot。

Runner選択:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order_rank ASC
4. source_order ASC
5. ready_at ASC
6. job_run_id

Different expansion/template間でもorder_rankは同じ整数軸として比較する。Job priorityをまたいでorder_byが優先することはない。

## 15. Template group status / conclusion

`needs: [evaluate]` 等で参照するTemplate groupは、そのtemplateに属する全expansion instance + generated Jobを集約する。

Status:

- activation/expansion未確定、またはgenerated Job non-terminalあり -> `running`
- 全expansion outcome確定 + generated Job全terminal -> `completed`

Conclusion優先順位:

1. cancel由来あり -> `cancelled`
2. expansion failure / non-allowed generated failureあり -> `failure`
3. required generated blockedあり -> `blocked`
4. 全expansion instanceが`skipped`でgenerated Job 0 -> `skipped`
5. それ以外で全generated Jobがeffective success/skipped、または正常empty expansion -> `success`

Nested parent Job 0件の場合は§7の直接propagationを優先する。

例:

- Root `if=false` -> group `completed/skipped`
- Root `foreach=[]` -> group `completed/success`
- Nested parent success+0 jobs -> nested `completed/success`
- Nested parent skipped+0 jobs -> nested `completed/skipped`
- Nested parent failure+0 jobs -> nested `completed/blocked`
- Nested parent10件の全てで`if=false` -> group `completed/skipped`
- 一部parentでskip、一部でsuccess -> group `completed/success`（failure/blocked/cancel無し）

このgroup conclusionを`02`のcondition helperが通常dependencyと同様に使う。

## 16. Output / Artifact aggregation

```text
needs.<template>.jobs[full_job_key]
needs.<template>.outputs[full_job_key]
needs.<template>.artifacts[full_job_key]
```

raw keyだけでindexしない。

Generated Job 0件なら各mapはempty object。

## 17. Retry / Recovery

Generated Job Retryはsame Job Run new Attempt。

固定:

- full/raw key
- item/iteration
- persistent Input
- order rank
- Action/Validator version

foreach/order再評価無し。

Recovery:

- committed expansion再生成無し
- uncommittedはcurrent dependenciesから再評価
- committed outcome (`expanded|skipped|failed|cancelled`)を重複確定しない
- zero-parent direct propagationもidempotent
- success generated Job保持
- running Runner recovery
- queued維持

## 18. Reusable / External / Human

Dynamic templateにReusable/External/Human可。Field/Validator constraintsは`01`。

External generated Jobもoptional Validator可。Human/Reusable generated JobはValidator禁止。

## 19. Pause / Cancel

Pause中new expansion無し。commit済み保持。

Cancel後new expansion無し。未実行generatedは通常cancel。

## 20. Failure code

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

## 21. 受入条件

1. Root/Nested/3+ depth
2. parent cycle
3. parent+needs helper
4. parent別same raw key
5. no fixed job_key limit
6. 1000/1001 rollback
7. Action+Validator version preflight before insert
8. atomic recovery
9. `order_by` source_order/list schema/asc/desc/stable tie
10. Root `if=false` group skipped
11. Root `foreach=[]` group success
12. nested parent success+0 -> success
13. nested parent skipped+0 -> skipped
14. nested parent failure/blocked+0 -> blocked
15. nested parent cancelled+0 -> cancelled
16. no parent item synthesis by always()
17. mixed skipped/success group success
18. group failure/blocked/cancel precedence
19. full-key output/artifact
20. Retry Action+Validator/item/order snapshot固定
