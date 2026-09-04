# 05. Dynamic Jobs 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/01-workflow-definition.md`
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/03-runtime-and-scheduling.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner における Dynamic Job の生成、識別、依存関係、順序、再開、失敗時の扱いを定義する。

対象:

- `foreach`
- `key`
- `item`
- `iteration`
- Dynamic Job template と生成 Job の関係
- Dynamic Job の入れ子
- 生成 Job 数上限
- Atomic expansion
- 生成 Job ID
- 重複 key / name collision
- `order_by`
- Dynamic Job 群への `needs`
- Output 集約
- Retry / Resume / Recovery
- Dynamic Job と Reusable Workflow の組み合わせ

## 2. 基本原則

1. Dynamic Job は YAML に定義された Job template を Engine が展開して生成する。
2. Action から Runtime API を使って任意 Job を直接追加する方式は MVP では採用しない。
3. `foreach` の評価結果は配列を基本とする。
4. 生成 Job は通常 Job と同じ Job Run / Attempt / Log / Artifact / Retry / status を持つ。
5. Dynamic Job の入れ子段数には固定上限を設けない。
6. 暴走防止は Workflow Run 全体の生成 Job 総数上限で制御する。
7. MVP の既定上限は 1000 Job とする。
8. 展開は all-or-nothing とし、途中まで Job を生成した状態を残さない。
9. 生成 Job の識別には stable key を優先する。
10. 同一 Workflow Run 内では Dynamic Job も含め internal Job の同時実行は最大 1 件とする。

## 3. YAML 基本形

```yaml
jobs:
  generate:
    action: generate_candidates

  evaluate:
    needs: [generate]
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    action: evaluate_candidate
    with:
      candidate: ${{ item }}
```

`evaluate` は Dynamic Job template であり、実行時に以下のような生成 Job へ展開される。

```text
evaluate[candidate_a]
evaluate[candidate_b]
evaluate[candidate_c]
```

Template 自体は Action を実行する Job Run として扱わず、生成 Job 群の論理的な親として扱う。

## 4. `foreach`

### 4.1 評価タイミング

`foreach` は以下を満たした時点で評価する。

1. `needs` に指定された依存 Job が必要条件を満たしている
2. Workflow Run が cancelled ではない
3. 必要な upstream Output / Artifact / state が参照可能
4. `if` がある場合、template activation 条件を満たしている

### 4.2 許容結果

MVP では `foreach` の評価結果として JSON array を要求する。

例:

```json
[
  {"id": "a", "priority": 10},
  {"id": "b", "priority": 5}
]
```

`null`、object、scalar は expansion error とする。

空配列は正常とする。

空配列時:

- 生成 Job 数は 0
- template は展開済みとして扱う
- template 群に依存する後続 Job は「0 件完了済み」として先へ進める

## 5. `item`

各生成 Job は `foreach` の各要素を `item` として snapshot する。

```yaml
with:
  candidate: ${{ item }}
  candidate_id: ${{ item.id }}
```

`item` は生成 Job 作成時に固定し、Retry 時も同じ値を使用する。

Source array が後から変化しても、既に生成済み Job の `item` は変更しない。

## 6. Stable key

### 6.1 `key`

Dynamic Job は可能な限り明示的な `key` を持つ。

```yaml
key: ${{ item.id }}
```

`key` の評価結果は string または integer とする。

内部では string へ canonical conversion して扱う。

空文字は不正。

### 6.2 生成 Job ID

概念上の生成 Job ID:

```text
<template_job_id>[<key>]
```

例:

```text
evaluate[candidate_123]
```

DB の primary key は別の opaque ID を使ってよいが、Workflow Run 内の論理 Job identity と表示用 reference は上記 stable key に基づく。

### 6.3 key 重複

同一 template の 1 回の expansion で duplicate key が発生した場合、展開全体を failure とする。

例:

```text
a
a
b
```

は不正。

途中まで生成しない。

### 6.4 key 未指定

`key` を省略した場合、source array index を fallback key として使用できる。

例:

```text
evaluate[0]
evaluate[1]
evaluate[2]
```

ただし index fallback は source array の並びが変わると Job identity も変わる。

静的検証では許可するが、可能なら明示 key を推奨する warning を出せるようにする。

## 7. Dynamic Job の入れ子

Dynamic Job からさらに Dynamic Job を生成できる。

例:

```text
evaluate[candidate_a]
└─ condition[x]
└─ condition[y]

condition[x]
└─ scenario[1]
└─ scenario[2]
```

入れ子深さに固定上限を設けない。

### 7.1 Context

子 Dynamic Job では現在の `item` に加え、親 iteration chain を参照できる。

概念 context:

```text
item
iteration.parent
iteration.ancestors
```

MVP の exact field 名は Expression 実装時に固定するが、少なくとも親要素を明示的に参照できる構造を提供する。

例:

```yaml
with:
  candidate: ${{ iteration.parent.item }}
  condition: ${{ item }}
```

### 7.2 Context snapshot

各生成 Job は以下を snapshot する。

- template job id
- stable key
- current item
- parent generated job reference
- ancestor iteration references
- source order
- calculated order key

Retry / Resume で再評価して別の iteration context に変化させない。

## 8. 生成 Job 数上限

### 8.1 既定値

```text
max_dynamic_jobs_per_workflow_run = 1000
```

この上限は Workflow Run 全体で数える。

入れ子階層ごとの個別上限ではない。

### 8.2 設定優先順位

想定優先順位:

1. Workflow 定義 override
2. System default

Job template 単位 override は MVP では必須にしない。

### 8.3 超過

新しい expansion を適用すると上限を超える場合、その expansion 全体を failure とする。

例:

- 既存 Dynamic Job: 950
- 新規候補: 100
- 上限: 1000

この場合 50 件だけ生成することは禁止する。

結果:

- 新規 100 件は 0 件生成
- expansion failure
- failure reason に current_count / requested_count / limit を含める

Silent truncate は禁止。

## 9. Atomic expansion

Dynamic Job 展開は必ず all-or-nothing で行う。

### 9.1 手順

1. `foreach` を評価
2. source array を snapshot
3. 各要素について `key` を評価
4. 各 Job candidate を memory 上で構築
5. 全 candidate を検証
6. Dynamic Job 総数上限を検証
7. 1 SQLite transaction で全 generated Job を登録
8. expansion marker / metadata を確定
9. commit

### 9.2 事前検証

少なくとも以下を全件検証する。

- duplicate key
- generated logical ID collision
- source value が JSON-compatible
- `with` expression 解決可能性
- `runs-on` が登録済み Runner Pool
- Action が Registry に存在
- Action version requirement
- executor の妥当性
- timeout / retry 値
- `if` / `order_by` expression compile
- 最大生成数
- parent / ancestor reference 整合

### 9.3 rollback

1 件でも validation / insert に失敗した場合 transaction を rollback する。

以下のような状態は禁止する。

```text
100 件予定
├─ 13 件登録済み
└─ 87 件未登録
```

## 10. Expansion identity / 再実行防止

同一 Workflow Run 内で同じ template / parent iteration に対する expansion を二重実行してはならない。

Core は少なくとも以下で expansion instance を識別する。

- workflow_run_id
- template_job_id
- parent_generated_job_id または root marker

Expansion 完了済みの場合、Scheduler が再度 activation を検出しても Job を重複生成しない。

Runtime crash が transaction commit 後に発生しても、再起動後は登録済み generated Job をそのまま使用する。

## 11. `order_by`

Dynamic Job は YAML で実行順を指定できる。

例:

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
  - expr: ${{ item.id }}
    direction: asc
```

### 11.1 未指定

未指定時は source array order を保持する。

### 11.2 `source_order`

明示的に source order を指定する簡易形式を許可してよい。

```yaml
order_by: source_order
```

### 11.3 Sort key

`order_by` の各式は generated Job 登録時に評価し、結果を snapshot する。

Job 実行直前に upstream state を見て再計算しない。

### 11.4 実行優先順位との関係

Runner が Job を選ぶ際の優先順位は以下。

1. Workflow Run priority
2. Job priority
3. Dynamic Job `order_by`
4. source order / ready order
5. queue wait time tie-break

Job priority は `order_by` より優先する。

## 12. Job priority

Dynamic Job template に指定された Job priority は、原則として生成 Jobへ継承する。

将来 item ごとの dynamic priority を許可してもよいが、MVP では `order_by` と固定 Job priority で十分とする。

## 13. `needs` と Dynamic Job 群

Dynamic Job template 名を `needs` に指定した場合、その template から生成された Job 群全体を指す。

```yaml
aggregate:
  needs: [evaluate]
```

### 13.1 完了判定

後続 Job が ready になるためには、`evaluate` の generated Job 群が依存条件を満たしている必要がある。

既定では:

- 全 generated Job が terminal
- failure が許容されていない場合は required Job が success

`continue-on-error` や `if: always()` は通常 Job と同じ規則に従う。

### 13.2 0 件 expansion

生成数 0 の template は依存関係上「完了済みの空集合」として扱う。

後続 Jobを永久待機させない。

## 14. Dynamic Job Output 集約

Template 名経由で generated Job 群の Output を参照できる。

```yaml
with:
  results: ${{ needs.evaluate.outputs }}
```

基本形は stable key を key にした map とする。

```json
{
  "candidate_a": {"score": 0.81},
  "candidate_b": {"score": 0.74}
}
```

### 14.1 個別参照

特定 generated Job の Output も参照可能にする。

概念例:

```text
needs.evaluate["candidate_a"].outputs
```

Exact expression syntax は Expression 詳細設計との整合を取って実装時に固定する。

### 14.2 Output 順序

Map を基本とするため stable key で参照する。

順序が必要な場合は generated Job metadata の `source_order` / `order_by_rank` を利用して list 化する helper を将来提供してよい。

## 15. Artifact 集約

Dynamic Job が登録した Artifact も generated Job 単位で保持する。

Template 経由の参照では stable key ごとの Artifact map として解決できる。

例:

```text
needs.evaluate.artifacts
```

概念結果:

```json
{
  "candidate_a": {
    "report": {"artifact_id": "...", "uri": "..."}
  },
  "candidate_b": {
    "report": {"artifact_id": "...", "uri": "..."}
  }
}
```

Artifact 実体の保存責任は通常 Job と同様に親システム / Action 側にある。

## 16. `if` と Dynamic Job

### 16.1 Template `if`

Template 自体に `if` がある場合、`foreach` 展開前に評価する。

False の場合:

- expansion しない
- template は skipped 相当
- generated Job は 0 件

### 16.2 Generated Job 単位 `if`

`if` が `item` を参照する場合、各 candidate ごとに評価する。

設計上は以下のどちらでも実装可能だが、MVP では candidate を生成した上で generated Job の `if` として保持し、通常 Scheduling ルールで skip 判定する方式を基本とする。

これにより Job 履歴上も「候補は存在したが条件で skipped」が確認できる。

## 17. Retry

Generated Job の Retry は通常 Job と同じ。

- 同じ Job Run
- 新しい Attempt
- `item` は変更しない
- stable key は変更しない
- parent iteration context は変更しない
- source order / order_by key は変更しない

Retry を理由に template 全体を再展開しない。

## 18. Resume

Workflow Run Resume 時、既存 generated Job を再利用する。

既に:

- success → そのまま success
- failed → Retry 操作 / policy 次第
- queued → そのまま queue
- running だったが Runner lost → Recovery policy
- waiting_external / waiting_review → 各 executor の状態に従う

成功済み Job を「skip したこと」に書き換えない。

## 19. Runtime restart / Recovery

Runtime 再起動時、Dynamic Job template ごとに expansion metadata を確認する。

### 19.1 expansion committed 済み

登録済み generated Job を正とする。

Source upstream Output を再評価して Job 群を作り直さない。

### 19.2 expansion transaction 未commit

Generated Job は 0 件のため、activation 条件を再評価して expansion を最初から実行できる。

### 19.3 部分登録

Atomic transaction を前提とするため、正常系では部分登録状態は存在してはならない。

DB corruption 等で不整合を検出した場合は fail-closed とする。

## 20. Source Output 変更時

同一 Workflow Run 内では upstream Job Output は terminal success 後の immutable result として扱う。

そのため一度 expansion が commit された後に source array が書き換わることは前提にしない。

Upstream Job を明示 Retry し、新しい Output が生じるケースでは既存 downstream expansion との整合が問題になるため、MVP では以下を原則とする。

- downstream generated Job がまだ存在しない場合のみ新 Output から展開可能
- 既に downstream expansion 済みの場合、上流 Job の手動 Retry は dependent Job を含む再実行範囲の整合確認を必要とする

Exact dependent reset policy は `10-retry-recovery-cancel.md` で定義する。

## 21. Reusable Workflow との組み合わせ

Dynamic Job の各 generated Job が Reusable Workflow を呼び出すことを許可する。

例:

```yaml
jobs:
  per_candidate:
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    uses: workflows/evaluate-candidate.yaml
    with:
      candidate: ${{ item }}
```

各 generated Job は独立した Child Workflow Run を起動する。

Dynamic Job の入れ子と Reusable Workflow の入れ子は別概念として管理する。

Circular Reusable Workflow reference は通常どおり拒否する。

## 22. External LLM / Human Job の Dynamic 展開

Dynamic Job の generated Job は executor に `external_llm` / `human` を指定できる。

例:

```yaml
review:
  foreach: ${{ needs.generate.outputs.items }}
  key: ${{ item.id }}
  executor: human
```

Generated Job は個別に:

- waiting_external
- waiting_review

へ遷移する。

同一 Workflow Run 内で external / human waiting は Runner を保持しない。

## 23. Cancel

Workflow Run Cancel 時:

- queued generated Job → cancelled
- waiting_external → cancelled
- waiting_review → cancelled
- running internal generated Job → graceful cancel request
- 未展開 template → 以後展開しない

Cancel 後に遅延した expansion callback が来ても generated Job を新規登録してはならない。

## 24. Failure reason

Dynamic Job expansion failure は structured failure として記録する。

代表 code:

```text
dynamic_foreach_type_invalid
dynamic_key_invalid
dynamic_key_duplicate
dynamic_job_id_collision
dynamic_job_limit_exceeded
dynamic_expression_error
dynamic_action_not_found
dynamic_runner_pool_not_found
dynamic_parent_context_invalid
dynamic_expansion_inconsistent
```

`details` には必要に応じて:

- template_job_id
- parent_generated_job_id
- candidate_index
- key
- current_count
- requested_count
- limit
- expression path

を含める。

## 25. Persistence requirements

Persistence 層は Dynamic Job 用に少なくとも以下を保持できる必要がある。

Generated Job metadata:

- workflow_run_id
- template_job_id
- generated_job_id
- stable_key
- parent_generated_job_id nullable
- item_snapshot_json
- iteration_context_json
- source_order
- order_rank / sort key
- expansion_id

Expansion metadata:

- expansion_id
- workflow_run_id
- template_job_id
- parent_generated_job_id nullable
- source_snapshot or source digest
- generated_count
- status
- created_at

Exact table / column は `08-persistence.md` で確定する。

## 26. Event Log

代表 event:

```text
dynamic_expansion_started
dynamic_expansion_completed
dynamic_expansion_failed
dynamic_job_generated
```

大量 Job 生成時に `dynamic_job_generated` を1000件 Event として必ず残すか、まとめ event にするかは Logging 詳細設計で決める。

少なくとも expansion 単位の started / completed / failed は必須。

## 27. Observability

Service / MCP / Web から以下を確認可能にする。

- template job id
- generated count
- generated Job list
- stable key
- source order
- order rank
- parent / ancestor relation
- individual status / conclusion
- Output / Artifact
- expansion failure reason

WebUI の表示方式そのものは別途設計する。

## 28. 静的検証

Workflow load 時に少なくとも以下を検証する。

- `foreach` が許可された位置にある
- `key` は `foreach` と同時使用される
- `order_by` は Dynamic Job のみ
- CEL / JMESPath compile
- `item` を使えない scope で参照していない
- obvious circular static dependency がない
- Dynamic Job template ID が通常 Job ID と衝突しない

実データ依存の duplicate key / 上限超過等は Runtime expansion 時に検証する。

## 29. 受入条件

MVP は少なくとも以下のテストを通す。

1. 配列3件から3 generated Jobを生成できる
2. Stable key が Job identity に反映される
3. duplicate key で 0 件登録のまま失敗する
4. 1000件までは生成できる
5. 1001件になる expansion は全件 rollback される
6. 空配列から0件 expansionが正常完了する
7. key 未指定時にindex fallbackできる
8. source orderが保持される
9. `order_by` で順序を変えられる
10. Job priorityが`order_by`より優先される
11. Dynamic Job templateへの`needs`がgenerated Job群全体を待つ
12. Outputがstable key mapとして集約される
13. 特定generated Jobを個別参照できる
14. 2段以上のnested Dynamic Jobを生成できる
15. 深さに固定上限がない
16. Workflow Run総生成数上限が入れ子を跨いで適用される
17. expansion commit後のRuntime crashで重複生成しない
18. Retryでtemplate再展開しない
19. success済みgenerated JobがResumeで再実行されない
20. External LLM / Human executorをDynamic Jobで利用できる
21. Cancel後に新規Dynamic Jobを生成しない
22. Reusable WorkflowをDynamic Jobとして展開できる

## 30. MVP外

以下はMVP必須としない。

- Actionからの任意Dynamic Job直接生成API
- Infinite stream / generatorを逐次Job化する機能
- 上限超過時の自動分割実行
- 分散環境でのDynamic expansion
- Dynamic Job専用GUI editor
- source array差分を使った既存generated Job群の自動再構成

これらが必要になった場合も、既存のstable key / expansion / generated Job modelを拡張して対応する。
