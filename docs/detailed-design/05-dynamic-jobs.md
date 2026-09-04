# 05. Dynamic Jobs 詳細設計

- Status: Draft v1.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `10`

## 1. 目的

`foreach` によるDynamic Job生成、任意深さの入れ子、stable key、full logical `job_key`、atomic expansion、ordering、dependency group、Recovery、Manual Retry後のexpansion reuseを定義する。

## 2. 基本原則

1. Actionから任意Jobを直接追加しない。YAML templateをEngineが展開する。
2. Dynamic templateは**仮想group**であり、それ自体の`job_runs`/Attempt/Runner実行を作らない。
3. Expansionで生成されたconcrete Jobだけが通常Job Run/Attempt/Retry/Log/Artifact/Validator規則を持つ。
4. 入れ子深さに固定上限無し。
5. Workflow Run generated Job数上限default1000。
6. 1 expansion all-or-nothing。
7. identity=parent path込みfull `job_key`。
8. 同一Run internal Job同時最大1。
9. `foreach=[]`正常0件と`if=false` skipを区別。
10. 一度`expanded`としてcommitしたgenerated Job集合は同一Run内で変更しない。
11. Dynamic expansionに使うSystem由来設定はcurrent System configではなくWorkflow Run開始時にsnapshot済み`effective_settings`を使う。

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

Template自身はAction/Validatorを実行しない。Run start時にTemplate用`job_runs` rowを作らず、Definition snapshot + `dynamic_expansions` + generated Job Runsからgroup stateを導出する。

Template IDそのものを`wf_retry`の`job_run_id`として指定できない。Expansion failureはAttempt無しfailure。

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

Each Root/Nested expansion instance:

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

Explicit `key`推奨。Duplicate raw keyはsame expansion instance内failure。

`key`省略時はsource array index（0-based integerのbase10 string）をraw keyとしてfallbackする。

Index fallbackはarray reorderでlogical identityが変わるため、**各expansion instanceで1回** structured Eventを記録する。

```text
type = dynamic_index_key_fallback
payload = {
  template_id,
  expansion_id,
  parent_generated_job_run_id,
  generated_count
}
```

Eventはwarning相当のobservabilityでありexpansionをfailureにはしない。Explicit keyを使った場合は発行しない。

Full key:

```text
evaluate[candidate_a]
evaluate[candidate_a]/condition[x]
```

Raw key UTF-8 percent-encode。Safe=`A-Z a-z 0-9 . _ -`、他uppercase `%HH`。

Full `job_key` fixed byte limit無し。DB TEXT、filesystem pathはopaque ID。

## 10. 生成数上限

Effective max=`workflow_run.effective_settings.max_dynamic_jobs` snapshot。

Root RunではSystem baseline + Workflow override、Child Runではinherited baseline + Child Workflow overrideからRun作成時に確定済み。

Canonical default=1000。

Count=当該Workflow Run generated **concrete Job**総数。Dynamic template group自体はcountしない。Child Runは別count。

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
- generated internal Jobのみresolved Runner Poolが登録済みか
- Action current ID/version == Run snapshot
- Validator current ID/version == Run snapshot
- Reusable prerequisites
- Retry/internal timeout
- Run snapshot `max_dynamic_jobs` limit

1件でも失敗ならgenerated Jobを残さない。

Commit transaction:

- expansion row/outcome
- source snapshot/digest
- expansion digest
- all generated concrete `job_runs`
- item/iteration/source_order/order_rank
- index fallback使用時のEvent

をall-or-nothingで保存する。

## 13. Expansion digest

`expanded` outcomeごとにManual Retry同一性確認用 `expansion_digest` を保存。

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

canonical-json-v1 -> SHA-256。Source=arrayそのもの、candidates=source order順。Empty arrayもdigestを持つ。

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

Allowed=`omitted|null|source_order|non-empty array {expr,direction}`。

Direction=`asc|desc`, default asc。Unknown key reject。

Criterion=stringまたはfinite number、criterion内同型。null/bool/object/array/NaN/Infinity/混在禁止。

Sort criteria left-to-right、tie=source_order ASC。`order_rank=0..N-1` snapshot。

Runner ordering=Workflow priority -> Job priority -> order_rank -> source_order -> ready_at -> id。

## 16. Template group status / conclusion

Template groupは仮想viewとして導出する。

Status:

- expansion未確定/non-terminal generatedあり -> running
- 全expansion確定 + generated全terminal -> completed

Conclusion priority:

1. cancel -> cancelled
2. expansion/non-allowed generated failure -> failure
3. required blocked -> blocked
4. all expansion skipped + generated0 -> skipped
5. その他effective success/skippedまたは正常empty -> success

Nested zero-parentは§7優先。

Generated Job 0件ならoutputs/artifacts/jobs map=empty object。

## 17. Output / Artifact aggregation

```text
needs.<template>.jobs[full_job_key]
needs.<template>.outputs[full_job_key]
needs.<template>.artifacts[full_job_key]
```

Raw keyだけでindexしない。

## 18. Generated Job Retry

Generated Job Retry=same Job Run new Attempt。

固定:

- full/raw key
- item/iteration
- persistent Input
- Secret bindings
- order rank
- Action/Validator version

foreach/key/order再評価無し。

## 19. Manual Retry後のExpansion Reuse

Upstream Manual Retry後、既存`expanded` expansionのgenerated集合を勝手に差し替えない。

Current dependency contextから副作用無し再計算:

1. expansion instance identity集合
2. current `if`
3. foreach source
4. raw/full key
5. item
6. source/order rank
7. preflight（Run snapshot settings）
8. expansion digest

Exact matchのみexisting expansion保持。各successful generated Jobは`03` strict Result Reuseへ。

Mismatch/error=`dynamic_expansion_not_reusable`, retryable=false, new Workflow Run要求。

### Whole skipped group

Group全体completed/skipped + generated0は未実行扱い。Manual Retry descendant reactivation時にskipped expansion rowをreset/removeしてcurrent dependenciesから再評価可。Past skipはEventへ。

### Expanded empty

`foreach=[]` はexpanded/success。Current digest exact match必須。Nonemptyへ変化したらnot reusable。

### Mixed successful group

一部skipped+一部expandedでgroup successの場合、group全体をsuccessfulとして扱い、skipを部分再展開しない。Current outcome/digest構造exact match必須。

## 20. Recovery

- committed expansion再生成無し
- uncommittedはcurrent dependenciesから再評価
- expansion digest復元
- zero-parent idempotent
- generated concrete Jobのrunning/queued recovery
- current System configを再参照せずRun snapshot settings継続
- `reuse_check_pending`時§19再開

## 21. Reusable / External / Human

Dynamic templateにReusable/External/Human可。生成されたconcrete Jobのfield constraintsは`01`。

## 22. Pause / Cancel

Pause中new expansion無し。Commit済み保持。

Cancel後new expansion無し。未実行generated Jobは通常cancel。

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

1. Dynamic template has no job_runs/Attempt/Retry target
2. Root/Nested/3+ depth
3. parent cycle/zero-parent
4. parent別same raw key/full key
5. index fallback key exact + one warning Event/expansion
6. explicit key no fallback Event
7. generated max from Run snapshot, template groups excluded
8. System setting change after Run start does not change limit
9. 1000/1001 rollback
10. internal-only Runner Pool preflight
11. Action/Validator preflight
12. atomic expansion/recovery
13. order schema/stable tie
14. if=false skipped vs foreach=[] success
15. group precedence
16. expansion_digest golden
17. Manual Retry exact expansion reuse
18. changed source/key/order/item -> new Run
19. whole skipped group re-evaluate only
20. mixed group no partial re-expansion
21. full-key aggregation
22. Generated Retry snapshot fixed
