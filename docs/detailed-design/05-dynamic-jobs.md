# 05. Dynamic Jobs 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `10`

## 1. 目的

`foreach` によるDynamic Job生成、任意深さの入れ子、stable key、full logical `job_key`、atomic expansion、ordering、dependency group、Recovery、Manual Retry後のexpansion reuseを定義する。

## 2. 基本原則

1. Actionから任意Jobを直接追加しない。YAML templateをEngineが展開する。
2. generated Jobは通常Jobと同じJob Run/Attempt/Retry/Log/Artifact/Validator規則。
3. 入れ子深さに固定上限無し。
4. Workflow Run generated Job数上限default1000。
5. 1 expansion all-or-nothing。
6. identity=parent path込みfull `job_key`。
7. 同一Run internal Job同時最大1。
8. `foreach=[]`正常0件と`if=false` skipを区別。
9. 一度`expanded`としてcommitしたgenerated Job集合は同一Run内で変更しない。

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

- parent=same Workflow Dynamic template ID literal
- parentはDAG edge/cycle対象
- parentをneedsへ重複記載禁止
- needsにはparent以外global dependency可
- condition dependency set=`{parent} ∪ needs`
- 1 parent generated Jobにつき0/1 expansion instance
- cancel後new expansion無し

Root scalar/Nested object form以外reject。

## 5. Expansion outcome

各Root/Nested expansion instance:

```text
pending|expanded|skipped|failed|cancelled
```

- `if=false` -> skipped
- foreach評価成功 + array commit -> expanded
- expression/preflight/storage error -> failed
- Workflow cancelで未展開 -> cancelled

`skipped` と empty `expanded` を同一扱いしない。

## 6. `foreach` result

JSON array必須。null/object/scalar -> `dynamic_foreach_type_error`。

Empty array:

- expansion=expanded
- generated count=0
- 正常な仕事なし
- Root group success

Source arrayはexpansion時snapshot。

## 7. Nested zero-parent propagation

Nested parent group terminal時にgenerated parent Job=0ならper-parent expressionを評価できないため、parent group conclusionを直接伝播する。

| parent group | nested group |
| --- | --- |
| success | success |
| skipped | skipped |
| failure | blocked |
| blocked | blocked |
| cancelled | cancelled |

Global needsも合わせ、優先方向は `cancelled > blocked/failure > skipped > success`。

`always()`でも存在しないparent itemを生成しない。

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

Ancestors outermost -> parent。Retryでitem/iterationを再評価しない。

## 9. raw key / full job key

Key result=string|integer。Integerはbase10 string。Empty string禁止。

Duplicate raw keyはsame expansion instance内failure。Key omitted=source index fallback。

```text
evaluate[candidate_a]
evaluate[candidate_a]/condition[x]
```

Raw key UTF-8 percent-encode。Safe=`A-Z a-z 0-9 . _ -`、他uppercase `%HH`。

Full `job_key` fixed byte limit無し。DB TEXT、filesystem pathはopaque ID。

## 10. 生成数上限

Effective max:

```text
Workflow settings.max-dynamic-jobs > System max_dynamic_jobs > 1000
```

Count=当該Workflow Run generated Job総数。Child Runは別count。

超過時当該expansion全rollback。Silent truncate禁止。

## 11. Expansion identity

```text
workflow_run_id
template_id
parent_generated_job_run_id nullable
```

Root parent=null、Nestedはparent generated Job。

## 12. Atomic expansion preflight

全candidateをmemory上で構築しDB insert前に全件検証:

- foreach result
- raw key type/duplicate
- full job_key collision
- parent/DAG
- Input/if/order expression validity
- executor field constraints
- Runner Pool
- Action current ID/version == Run snapshot
- Validator current ID/version == Run snapshot
- Reusable prerequisites
- Retry/internal timeout
- generated count limit

1件でも失敗ならgenerated Jobを残さない。

Commit transaction:

- expansion row/outcome
- source snapshot/digest
- expansion digest
- all generated `job_runs`
- item/iteration/source_order/order_rank

をall-or-nothingで保存。

## 13. Expansion digest

`expanded` outcomeごとに、Manual Retry後の同一性確認用 `expansion_digest` を保存する。

Canonical source:

```json
{
  "template_id": "evaluate",
  "parent_generated_job_run_id": null,
  "source": [],
  "candidates": [
    {
      "raw_key": "a",
      "full_job_key": "evaluate[a]",
      "item": {},
      "source_order": 0,
      "order_rank": 0
    }
  ]
}
```

をcanonical-json-v1 -> SHA-256。

`source`にはforeach評価結果arrayそのもの、`candidates`には全candidateをsource order順で入れる。

Empty arrayもdigestを持つ。

## 14. `if`

Root=declared needs terminal後1回。Nested=parent generated Job + declared needs terminal後parentごと。

- true -> foreach
- false -> expansion skipped/generated0
- error -> expansion failed

Zero-parentは§7。

## 15. `order_by`

Omitted=`source_order`。

```yaml
order_by: source_order
```

または:

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
  - expr: ${{ item.id }}
    direction: asc
```

Allowed:

```text
omitted|null
literal source_order
non-empty array of {expr,direction}
```

Direction=`asc|desc`, default asc。Unknown key reject。

Criterionはstringまたはfinite number、criterion内同型。null/bool/object/array/NaN/Infinity/混在禁止。

Sort criteria left-to-right、tie=source_order ASC。`order_rank=0..N-1` snapshot。

Runner orderingはWorkflow priority -> Job priority -> order_rank -> source_order -> ready_at -> id。

## 16. Template group status / conclusion

Status:

- expansion未確定/non-terminal generatedあり -> running
- 全expansion確定 + generated全terminal -> completed

Conclusion priority:

1. cancel -> cancelled
2. expansion/non-allowed Job failure -> failure
3. required blocked -> blocked
4. all expansion skipped + generated0 -> skipped
5. その他effective success/skippedまたは正常empty -> success

Nested zero-parentは§7優先。

Generated Job 0件ならoutputs/artifacts/jobs mapはempty object。

## 17. Output / Artifact aggregation

```text
needs.<template>.jobs[full_job_key]
needs.<template>.outputs[full_job_key]
needs.<template>.artifacts[full_job_key]
```

Raw keyだけでindexしない。

## 18. Generated Job Retry

Generated Job Retryはsame Job Run new Attempt。

固定:

- full/raw key
- item/iteration
- persistent Input
- Secret bindings
- order rank
- Action/Validator version

foreach/key/orderは再評価しない。

## 19. Manual Retry後のExpansion Reuse

Upstream JobをManual Retryした場合、descendantに**既にcommit済み`expanded` expansion**があるなら、そのgenerated集合を勝手に差し替えない。

副作用無しでcurrent dependency contextから再計算:

1. expansion instance identity集合
2. current `if`
3. foreach source array
4. raw/full key
5. item snapshot
6. source order/order rank
7. Action/Validator/Runner Pool preflight
8. canonical `expansion_digest`

Required:

- existing `expanded` instanceの集合が同一
- 全instanceでcurrent `if=true`
- current expansion digest == stored digest

Exact matchならexisting expansion/generated Jobを保持し、その後各successful generated Jobは`03` Result Reuse check。

Mismatch/error:

```text
category=runtime
code=dynamic_expansion_not_reusable
retryable=false
```

としてsame Runをfailureにしnew Workflow Runを要求する。

### 19.1 skipped expansion

Template group全体が`completed/skipped`でgenerated Job=0の場合は未実行扱いなので、Manual Retry descendant reactivation時にそのtemplateの`skipped` expansion rowをreset/removeしてcurrent dependenciesから再評価してよい。過去skip事実はEvent Logへ残す。

### 19.2 expanded empty

`foreach=[]` は`expanded/success`なので実行済み成功group。Manual Retry後もcurrent digest exact match必須。新たに非emptyへ変わった場合はsame RunでJob生成せず `dynamic_expansion_not_reusable`。

### 19.3 mixed group

一部expansion skipped + 一部expandedでgroup successの場合は**group全体をsuccessfulとして扱う**。Skipped instanceも含めcurrent outcome/digest構造が同一でなければreuse不可。Group successを理由に一部skipだけ再展開しない。

## 20. Recovery

- committed expansion再生成無し
- uncommittedはcurrent dependenciesから再評価
- committed outcome重複確定無し
- expansion digest復元
- zero-parent propagation idempotent
- running Runner recovery
- queued維持
- `reuse_check_pending`時は§19再開

## 21. Reusable / External / Human

Dynamic templateにReusable/External/Human可。Field/Validator constraintsは`01`。

## 22. Pause / Cancel

Pause中new expansion無し。Commit済み保持。

Cancel後new expansion無し。未実行generatedは通常cancel。

## 23. Failure code

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
dynamic_expansion_not_reusable
```

## 24. 受入条件

1. Root/Nested/3+ depth
2. parent cycle/zero-parent propagation
3. parent別same raw key
4. no fixed job_key limit
5. System/Workflow 1000 hierarchy + 1001 rollback
6. Action/Validator version preflight
7. atomic expansion/recovery
8. order schema/asc/desc/stable tie
9. Root if=false skipped vs foreach=[] success
10. group precedence/mixed skip-success
11. expansion_digest golden
12. Manual Retry exact expansion reuse
13. changed source/key/order/item -> not reusable/new Run
14. expanded empty changed to nonempty -> not reusable
15. whole skipped group may re-evaluate
16. mixed successful group cannot partially re-expand
17. full-key output/artifact
18. Generated Retry snapshot固定
