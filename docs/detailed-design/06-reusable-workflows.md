# 06. Reusable Workflows 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/01-workflow-definition.md`
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/03-runtime-and-scheduling.md`
  - `docs/detailed-design/05-dynamic-jobs.md`

## 1. 目的

本書は JobRunner における Reusable Workflow の定義、呼び出し、入力・出力、親子 Workflow Run の状態連携、Retry / Cancel / Recovery を定義する。

## 2. 基本原則

1. Reusable Workflow は MVP から対応する。
2. 親 Workflow から見た Reusable Workflow は 1 Job として扱う。
3. 子 Workflow は独立した Workflow Run を持つ。
4. 親と子の mutable state は共有しない。
5. データ受け渡しは Input / Output / Artifact の明示 mapping のみとする。
6. 子 Workflow の定義も Workflow Run 開始時に snapshot する。
7. 循環参照は開始前または展開時に拒否する。
8. 深さ自体には固定上限を必須としない。

## 3. YAML 定義

Job は `uses` で Reusable Workflow を参照する。

```yaml
jobs:
  analyze:
    uses: workflows/analyze.yml
    with:
      document_id: ${{ inputs.document_id }}
```

`action` と `uses` は相互排他とする。

以下も同時指定不可とする。

- `action`
- `uses`
- `executor: external_llm`
- `executor: human`

Reusable Workflow Job では `runs-on` は指定しない。Runner 選択は子 Workflow 内の各 Job が行う。

## 4. Workflow reference

MVP では親システムが解決可能な Workflow ID または相対 path を許可する。

例:

```yaml
uses: workflows/common/analyze.yml
```

または:

```yaml
uses: common.analyze
```

canonical な解決方法は `WorkflowResolver` に集約する。

Workflow YAML 自身が任意 URL を取得する機能は MVP に入れない。

## 5. Input

親 Job の `with` で子 Workflow Input を構築する。

```yaml
jobs:
  child:
    uses: workflows/child.yml
    with:
      symbol: ${{ inputs.symbol }}
      candidates: ${{ needs.generate.outputs.candidates }}
```

子 Workflow Input は通常の Workflow Input Schema で検証する。

検証失敗時は子 Workflow Run を作成せず、親 Job を failure とする。

## 6. Child Workflow Run 作成

親 Reusable Workflow Job が ready になったとき、Runtime は以下を atomic に実施する。

1. 子 Workflow Definition を解決
2. 循環参照検証
3. 子 Workflow Input を評価
4. 子 Definition / Action / Runner Pool を再検証
5. 子 Workflow Run を作成
6. 親 Job Run と子 Workflow Run の relation を保存
7. 子 Workflow の root Job を activation 対象にする

親 Job の Attempt ごとに子 Workflow Run は 1 つだけ作る。

## 7. 親子識別

子 Workflow Run は少なくとも以下を持つ。

```text
parent_workflow_run_id
parent_job_run_id
parent_attempt_id
root_workflow_run_id
call_depth
```

親を持たない通常 Workflow Run では `parent_*` は null。

## 8. Definition Snapshot

子 Workflow Run も通常 Workflow Run と同じく以下を snapshot する。

- Workflow YAML
- Definition hash
- Workflow version
- Input snapshot
- Action ID / version
- optional source_identity

親 Workflow Run の snapshot に子 Workflow の全内容を埋め込む必要はない。

親子 relation と、それぞれの独立 snapshot を保存する。

## 9. Workflow state

子 Workflow Run は独立した state namespace を持つ。

禁止:

```text
child -> parent mutable state direct set
parent -> child mutable state direct set
```

必要な値は Input として渡す。

子の計算結果は Output として返す。

## 10. Output

Reusable Workflow は Workflow-level Output を定義可能とする。

```yaml
outputs:
  score: ${{ jobs.aggregate.outputs.score }}
  report: ${{ jobs.report.artifacts.report }}
```

子 Workflow が success した時点で Workflow Output を評価する。

親 Job の Output として同じ値を公開する。

親側:

```yaml
with:
  score: ${{ needs.child.outputs.score }}
```

Artifact 参照も同様に明示 mapping する。

## 11. Conclusion propagation

子 Workflow Run の conclusion と親 Job の conclusion は原則連動する。

| Child conclusion | Parent Job conclusion |
| --- | --- |
| success | success |
| failure | failure |
| cancelled | cancelled |

子 Workflow 内の `continue-on-error` は子 Workflow 自身の最終 conclusion 算出に反映された後、親へ伝播する。

## 12. Retry

親 Job を Retry した場合、新しい Attempt を作成し、その Attempt に対して新しい子 Workflow Run を作る。

既存の子 Workflow Run は書き換えない。

```text
Parent Job
├─ Attempt 1
│  └─ Child Workflow Run A -> failure
└─ Attempt 2
   └─ Child Workflow Run B -> success
```

Retry Input は元親 Job Inputと同一とする。

## 13. Cancel

親 Workflow Run または親 Job が cancel された場合、実行中の子 Workflow Run へ cancel を伝播する。

子 Workflow Run の graceful cancel 規則は通常 Workflow Run と同じ。

子だけを cancel する管理 API を提供するかは Service API 詳細設計に委ねるが、MVP では親子整合性を優先し、親 Job を経由した cancel を基本とする。

## 14. Pause / Resume

親 Workflow Run が pause されても、すでに開始済みの子 Workflow Run内で実行中の Job は停止しない。

ただし親側で新規 Child Workflow Run を開始しない。

子 Workflow Run 自身の pause 状態は独立管理できる。

## 15. 循環参照

以下を禁止する。

```text
A -> A
A -> B -> A
A -> B -> C -> A
```

Workflow Definition load 時に静的解決できる範囲は事前検出する。

動的 resolver を用いる場合も Child Run 作成直前に ancestor chain を検査する。

cycle 検出時は `workflow_cycle_detected` failure とする。

## 16. 深さ

固定 depth limit は MVP の必須制約にしない。

ただし `call_depth` は保存し、将来 system setting で上限を導入できる構造にする。

## 17. Dynamic Job との組み合わせ

Reusable Workflow Job に `foreach` を付けることを許可する。

```yaml
jobs:
  evaluate:
    foreach: ${{ needs.generate.outputs.items }}
    key: ${{ item.id }}
    uses: workflows/evaluate-one.yml
    with:
      item: ${{ item }}
```

生成された各 Job が独立した Child Workflow Run を持つ。

Dynamic Job 総数上限 1000 は Reusable Workflow Job にも適用する。

子 Workflow 内でさらに Dynamic Job を生成してよい。

## 18. Concurrency

子 Workflow は自身の `concurrency` 設定を持てる。

親の concurrency group を自動継承しない。

必要なら親 Input から同じ値を子へ渡し、子 Definition 側で同じ group を構築する。

## 19. Authorization / Actor

子 Workflow Run は親 Workflow Run の Actor / AccessScope を引き継ぐ。

子の Service operation も AuthorizationProvider を必ず通す。

## 20. Event Log

少なくとも以下を記録する。

```text
child_workflow_requested
child_workflow_started
child_workflow_completed
child_workflow_cancel_propagated
child_workflow_cycle_rejected
```

Event には親 Job / Attempt と子 Workflow Run の ID を含める。

## 21. Recovery

Runtime 再起動後は親 Job と子 Workflow Run relation をDBから復元する。

- 子が running なら通常 recovery を継続
- 子が completed なら親 Job の未確定状態を再評価
- 親 Attempt が terminal なのに新しい子を生成しない
- 同一 parent Attempt に子 Workflow Run を重複生成しない

## 22. Failure code

代表 code:

```text
workflow_not_found
workflow_input_invalid
workflow_cycle_detected
child_workflow_start_failed
child_workflow_failed
child_workflow_cancelled
workflow_output_invalid
```

## 23. 受入条件

最低限以下をテストする。

1. 親 -> 子 Workflow success
2. 子 failure の親伝播
3. Input mapping
4. Output mapping
5. Artifact mapping
6. state 非共有
7. direct cycle 拒否
8. indirect cycle 拒否
9. Retry で新 Child Workflow Run
10. Cancel propagation
11. Runtime restart 後の relation 復元
12. Dynamic Job + Reusable Workflow
13. nested Reusable Workflow
14. Child concurrency
15. duplicate Child Run 防止
