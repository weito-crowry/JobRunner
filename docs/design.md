# JobRunner 設計書

- Status: Draft v0.1
- 対象: MVP 基本設計
- WebUI: 画面設計のみ別途実施
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

JobRunner は、既存アプリケーションへ組み込んで使う軽量な Persistent Workflow / Job Runtime である。

GitHub Actions の管理モデルを参考に、以下を提供する。

- Workflow / Workflow Run
- Job / Step / Attempt
- `needs` による依存関係
- Action Registry
- Runner / Runner Pool
- Retry / Resume / Pause / Cancel
- Workflow Run History
- Artifact 参照管理
- Human Review
- External LLM の pull 型実行
- Dynamic Job (`foreach`)
- Reusable Workflow
- Event / Execution Log
- SQLite による永続化
- MCP / Web から共通 Service API を操作

JobRunner 自体を Airflow / Temporal / n8n のような大規模な独立基盤にはしない。

## 2. 非目標

MVP では以下を行わない。

- GUI Workflow Editor / Node Editor
- SaaS Connector Marketplace
- 任意 Shell 実行を Core 標準機能として提供
- Remote arbitrary runner
- Multi-machine distributed scheduler
- Kubernetes scheduler
- CPU / RAM / GPU を考慮した高度な資源スケジューリング
- Cron / Scheduler
- CLI
- 通知機構（Slack / Mail 等）
- 独自 Expression DSL
- GitHub Actions 完全互換
- OS レベルの強制 Sandbox
- 認証・ユーザー管理
- 実装コード・依存パッケージ・Git dirty diff 等の完全な実行環境アーカイブ

## 3. 用語

| 用語 | 意味 |
| --- | --- |
| Workflow | YAML で記述する実行設計図 |
| Workflow Run | Workflow を 1 回起動した実行実体 |
| Job | Workflow 内の実行単位 |
| Attempt | Retry を含む Job の各試行 |
| Step | Job 内の観測・ログ・進捗単位。Scheduler 単位ではない |
| Action | 親システムが Action Registry に登録する実処理 |
| Runner | internal Job を実行する常駐 Process |
| Runner Pool | Runner のグループ |
| Artifact | Job が生成した成果物への参照情報 |

`Worker` ではなく `Runner` を正式用語とする。

## 4. 全体アーキテクチャ

```text
Parent System
├─ JobRunner Runtime / Service
├─ Action Registry
├─ SQLite
├─ MCP Adapter (optional)
├─ Web Adapter (optional)
└─ Runner Pools
   ├─ Runner Process
   │  └─ Action Runner Process
   └─ Runner Process
      └─ Action Runner Process
```

### 4.1 組み込み方式

- JobRunner は通常、親システムへ組み込んで使用する。
- JobRunner 専用サービスを利用者が別途手動起動する前提にはしない。
- 親システム起動時に Runtime と Runner Pool を初期化する。
- 親システム終了時に Runner を停止する。
- MCP / Web は Adapter とし、Core はそれらへ依存しない。

### 4.2 Python Package

1 Repository / 1 Python Package とする。

想定配布形:

```text
jobrunner
jobrunner[mcp]
jobrunner[web]
jobrunner[all]
```

想定構成:

```text
jobrunner/
├─ definitions/
├─ runtime/
├─ scheduler/
├─ runners/
├─ actions/
├─ expressions/
├─ persistence/
├─ artifacts/
├─ events/
├─ security/
├─ adapters/
│  ├─ mcp/
│  └─ web/
└─ migrations/
```

## 5. Workflow Definition

### 5.1 Canonical format

- Workflow の記述・保存形式は YAML。
- YAML はロード後に Schema 検証し、内部では typed immutable model として扱う。
- YAML に任意 Python / Shell コードを埋め込まない。
- 実処理は Action Registry に登録された Action を呼ぶ。

### 5.2 Workflow Run の Definition Snapshot

Workflow Run 開始時に、実際に使用した Workflow 定義を固定する。

保存対象:

- Workflow ID / version
- 元 YAML 全文
- 定義全体の hash
- Input snapshot
- 使用する Action ID / Action version
- optional `source_identity`

Workflow Run 開始後に元 YAML が変更されても、その Workflow Run には影響させない。

Retry / Resume も元 Workflow Run の snapshot を基準にする。

`source_identity` は親システムが任意に設定できる文字列であり、Git SHA に限定しない。

例:

- Git SHA
- package version
- build id
- Docker digest
- custom string

Core は `source_identity` の正しさを検証しない。

## 6. YAML 方針

GitHub Actions に対応する概念がある場合、可能な限り名称と読み方を寄せる。

例:

```yaml
name: Strategy Evaluation

inputs:
  symbol:
    type: string
    required: true

env:
  MODE: research

jobs:
  generate:
    runs-on: default
    action: fx.generate_candidates
    with:
      symbol: ${{ inputs.symbol }}

  evaluate:
    needs: [generate]
    runs-on: heavy
    foreach: ${{ needs.generate.outputs.candidates }}
    key: ${{ item.id }}
    action: fx.evaluate_candidate
    with:
      candidate: ${{ item }}
    retry:
      max-attempts: 3
      if: ${{ failure.retryable }}

  aggregate:
    needs: [evaluate]
    runs-on: default
    if: ${{ always() }}
    action: fx.aggregate
    with:
      results: ${{ needs.evaluate.outputs }}
```

`retry` / `foreach` 等、GitHub Actions に直接対応する構文がない機能は JobRunner 独自拡張とするが、キー名や式記法は全体として GitHub Actions に近づける。

## 7. Expression

独自 DSL は作らず、既存 OSS を利用する。

### 7.1 CEL

主用途:

- `if`
- `success_if`
- Retry 条件
- `foreach`
- `key`
- `order_by`
- Input mapping
- Concurrency key
- 値の簡単な計算

YAML 上の式表現は GitHub Actions 風の `${{ ... }}` を使用する。

### 7.2 JMESPath

複雑な JSON 抽出・絞り込み用として MVP から導入する。

CEL と役割分担し、必要に応じて CEL から `jmespath(...)` helper を呼べる形を提供する。

例:

```yaml
foreach: ${{ jmespath(needs.generate.outputs, 'candidates[?score > `0.8`]') }}
```

## 8. Action Registry

### 8.1 基本

親システムが `action_id -> callable` を登録する。

例:

```python
def setup_actions(registry):
    registry.register("fx.run_backtest", version="3", callable=run_backtest)
```

Action Registry 自体は DB へ永続化しない。

親システムに共通 bootstrap を用意し、以下から同じ bootstrap を使って Registry を再構築する。

- 親 Process
- Runner Process
- Action Runner Process

### 8.2 Action contract

単純な Action は engine-specific context を要求しない。

```python
def action(input_data) -> dict:
    ...
```

Runtime 連携が必要な Action は optional Runtime Handle を受け取れる。

Runtime Handle の用途:

- progress
- step
- execution log
- Workflow state get / set
- graceful cancel check
- Artifact 登録

Action は sync / async の両方を正式対応する。

### 8.3 Action version

Action は `action_id + version` で識別する。

- Workflow Run 開始時に version を記録する。
- Retry / Resume 時に要求 version と現在 Registry を照合する。
- 不一致は fail-closed とする。
- Action 実装変更時の version 更新は親システムの責任。
- Core は Python source 全体を解析・保存しない。

## 9. Runner / Runner Pool

### 9.1 Runner Pool

Runner Pool は親システム側で事前登録する。

YAML では GitHub Actions に合わせて `runs-on` を使う。

```yaml
runs-on: heavy
```

未登録 Pool は Workflow 開始前の検証でエラーにする。

MVP で Pool が持つ主要設定:

- name
- runner_count
- restart policy

Pool 単位の Action allow-list は持たない。

### 9.2 常駐方式

- Runner Process は親システム起動時に自動起動する。
- Job がない間も待機する。
- Job ごとに Runner 自体を起動・停止しない。
- 異常終了時は restart policy に従い再起動する。

### 9.3 Pull scheduling

Runner は Core から Job を pull する。

```text
Runner
  ↓
「自分の Pool で実行できる Job を 1 件取得」
  ↓
Core が atomic に Job を割当
  ↓
Runner が実行
  ↓
完了報告
  ↓
次の Job を pull
```

Runner が取得する単位は Workflow Run ではなく Job。

ただし Core は Job 選択時に Workflow Run 単位の制約を確認する。

### 9.4 1 Workflow Run = internal Job 同時実行最大 1

1 Workflow Run 内では internal Job を同時に複数実行しない。

- 同一 Workflow Run: internal Job は最大 1 件実行
- 別 Workflow Run: 複数 Runner で並列実行可能
- External LLM 待ち / Human Review 待ちでは Runner を保持しない
- 再開時は別 Runner が担当してよい

Runner は Workflow Run に固定しない。

### 9.5 Job 選択優先度

基本優先順位:

1. Workflow Run priority
2. Job priority
3. Dynamic Job の `order_by` 順位
4. 生成・ready 順
5. queue 待ち時間を最終 tie-break として利用

現在実行中 Job は priority 変更で preempt しない。

## 10. Runner と Action 実行 Process

FX-LLM で実績のある構造と同様に、Runner 管理 Process と実処理 Process を分ける。

```text
Runner Process
├─ Job pull
├─ Heartbeat Thread
├─ Job / Action Process 監視
└─ Common Action Runner Process
   └─ Action
```

利点:

- Runner 死亡と長時間 Action を区別できる
- Action 子 Process 異常終了を Runner が検知できる
- Action timeout / cancel を Process 単位で扱いやすい
- Heartbeat が Action 本体に阻害されない

## 11. Heartbeat / Runner Recovery

Runner Process 内で Heartbeat Thread を動かす。

MVP 既定値:

- Heartbeat interval: 5 秒
- Runner lost 判定: 最終 Heartbeat から 20 秒

両方とも設定で変更可能。

Runner lost 時:

1. Runner を lost と判定
2. 実行中 Attempt を `runner_lost` として確定
3. Job の Retry / Recovery policy に従う
4. Runner restart policy に従って Runner Process を再起動

親システム再起動時も同じ考え方で DB の `running` Job と生存 Runner を照合する。

Restart policy は少なくとも以下を設定可能にする。

- mode: `on_failure` / `never`
- max_restarts
- restart window
- backoff

Crash loop 時は自動再起動を抑止し、その状態を Service / MCP / Web から確認できるようにする。

## 12. Runner ↔ Action Runner IPC

ローカル IPC は JSON Lines を基本とする。

### 12.1 Runner → Action Runner

- Job Input
- Action ID / version
- Workflow Run / Job / Attempt 情報
- 一時作業ディレクトリ
- 必要な Workflow state
- Secrets
- cancel request

### 12.2 Action Runner → Runner

主な event:

- `log`
- `progress`
- `step_started`
- `step_finished`
- `artifact_registered`
- `result`
- `error`

Action の通常 stdout / stderr が protocol を破壊しないよう、Common Action Runner が捕捉して Execution Log へ転送する。

構造化 protocol は専用経路で扱う。

## 13. Job / Attempt / Step

### 13.1 Job status

外部表示は GitHub Actions に寄せ、`status` と `conclusion` を分ける。

主な `status`:

- `queued`
- `running`
- `waiting_external`
- `waiting_review`
- `paused`
- `completed`

主な `conclusion`:

- `success`
- `failure`
- `cancelled`
- `skipped`
- `blocked`

例:

```text
status = completed
conclusion = failure
```

`partial` は Job status として持たない。

許容可能な一部失敗は Workflow 定義の成功条件で表現し、Job 自体は success / failure のどちらかに確定する。

### 13.2 Attempt

Retry は既存 Attempt を書き換えず、新しい Attempt を追加する。

```text
Job
├─ Attempt 1 → failure
├─ Attempt 2 → failure
└─ Attempt 3 → success
```

Retry では Job Input を変更しない。

Input を変更する場合は同じ Retry ではなく、新しい実行として扱う。

### 13.3 Step

Step は観測単位。

- Scheduler 単位ではない
- `needs` を持たない
- Step 単独 Retry はしない
- Artifact ownership の単位にはしない
- Retry は Job 全体

## 14. Failure / Retry

### 14.1 Structured failure

Failure は少なくとも以下を持つ。

```text
category
code
message
retryable
details
```

代表 category / code:

- execution error
- validation error
- protocol error
- timeout
- `runner_lost`
- action version mismatch
- lease expired
- human rejected
- cancelled
- internal error

### 14.2 Retry

既定は manual retry。

Automatic retry は Job 単位で YAML 設定可能。

GitHub Actions に専用 Retry YAML がないため、JobRunner 独自拡張とするが記法は GitHub Actions 風にする。

例:

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
```

### 14.3 Timeout

Job execution timeout は optional。

- 未指定: timeout なし
- 指定時のみ有効
- timeout は failure とし、Retry policy を適用

External LLM lease timeout と Job execution timeout は別概念。

## 15. Pause / Resume / Cancel

### 15.1 Pause

Workflow Run 単位。

- 実行中 Job はそのまま継続
- 新しい Job を開始しない
- 新しい External task claim を許可しない
- 既に claim 済み External task の submit は受け付ける
- Human Review の結果送信は受け付ける
- Resume で scheduling を再開

### 15.2 Cancel

標準 Cancel は graceful cancel。

- queued / waiting Job は即 cancel
- 新しい Job は開始しない
- running internal Action へ cancel request
- late external submit は拒否

通常 Service API として force-kill は提供しない。

本当に hung した Process を kill / restart することは運用上の復旧手段として扱う。

## 16. Workflow failure control

GitHub Actions 風に `continue-on-error` を利用可能にする。

Cleanup / finally 専用機構は作らず、通常 Job + condition で表現する。

例:

```yaml
cleanup:
  if: ${{ always() }}
  action: system.cleanup
```

CEL helper として少なくとも以下を提供する。

- `success()`
- `failure()`
- `cancelled()`
- `always()`

## 17. Inputs / Outputs

### 17.1 Workflow Input

- typed schema
- required / optional / default
- Workflow Run 開始時に snapshot
- MCP / Web / Python API の全経路で同じ validator を使う

### 17.2 Job Input

以下を両方許可する。

1. Workflow Input / upstream output / Artifact から明示 field mapping
2. JSON object 全体を渡す

両方を組み合わせる場合、明示 mapping を優先できる。

最終 Job Input は Attempt 開始前に固定する。

### 17.3 Job Output

- JSON-compatible data
- Output Schema は optional
- 指定時は JSON Schema で検証
- 未指定なら JSON-compatible であれば受理

小さい JSON Output は Core が保持する。

大きいデータやファイルは Action が親システム側へ保存し、Artifact 参照を Output として返すことを基本とする。

Core が巨大 payload の永続保存方式を自動決定することは MVP の主責務にしない。

## 18. Workflow shared state

Workflow Run 内に共有値を持てる。

### 18.1 Static values

YAML `env` などの固定値。

Workflow Run 中は immutable。

### 18.2 Mutable state

Runtime Handle から `get / set` を提供。

Core が保証するのは:

- get
- set
- last-write-wins
- persistent across restart / resume

CAS / atomic increment は MVP に入れない。

すべての変更を append-only history に残す。

履歴項目:

- variable name
- old value
- new value
- revision
- timestamp
- Job
- Attempt
- Step (該当時)

Reusable Workflow の子 Workflow Run は独立 state を持つ。

## 19. Artifact

Artifact 実体の保存は親システム / Action の責任とする。

Core は Artifact の参照情報と履歴を管理する。

主な metadata:

```text
artifact_id
name
uri
media_type
size
digest (optional)
producer workflow_run / job / attempt
created_at
```

後続 Job は名前付きで参照できる。

```yaml
with:
  source: ${{ needs.export.artifacts.dataset }}
```

Core が渡すのは Artifact 実体ではなく参照情報。

実際の読み込み方法は Action / 親システム側の責任。

Artifact は論理的には immutable とする。

同名成果物を Retry で再生成しても過去 Artifact は履歴として残し、新しい Artifact を current として解決する。

## 20. Workflow Run directory

Workflow Run ごとの専用 directory を持つ。

用途は Core 自身のログと一時領域の整理に限定する。

```text
jobrunner-data/
└─ runs/
   └─ <workflow_run_id>/
      ├─ logs/
      │  └─ <job_id>/
      │     └─ <attempt>.log
      └─ tmp/
         └─ <job_id>/
            └─ <attempt>/
```

### 20.1 Temporary working directory

Job / Attempt 開始時に専用一時 directory を作成し、Action に渡す。

Attempt 終了時に削除する。

- success / failure / cancel を問わず原則削除
- 保存が必要なデータは Action が親側へ明示保存
- Core は一時 directory から保存対象を自動選別しない

この directory は security sandbox ではない。

## 21. Dynamic Job (`foreach`)

Dynamic Job は MVP から対応する。

Action が Runtime API で直接 Job を生成するのではなく、YAML の `foreach` を Engine が展開する。

例:

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

### 21.1 Nested expansion

Dynamic Job からさらに Dynamic Job を生成できる。

- 入れ子段数に固定上限を設けない
- 深さではなく動的生成 Job 総数で暴走防止する

### 21.2 最大生成数

既定値:

```text
max_dynamic_jobs_per_workflow_run = 1000
```

System / Workflow 設定で変更可能。

超過時は明示 failure。

Silent truncate は禁止。

### 21.3 Stable key

Dynamic Job には可能な限り stable key を要求する。

例:

```yaml
key: ${{ item.id }}
```

内部表示例:

```text
evaluate[candidate_123]
```

- duplicate key: expansion failure
- key 未指定時: array index fallback 可
- index fallback は source order が変わると同一性も変わることを明示する

### 21.4 Atomic expansion

展開は部分成功を残さない。

1. `foreach` を評価
2. 全 Job candidate を memory 上で構築
3. key 重複、name collision、最大数、input expression、Action 存在等を全件検証
4. SQLite transaction で一括登録
5. 1 件でも失敗したら rollback

### 21.5 Ordering

Dynamic Job は YAML で順序を指定可能。

例:

```yaml
order_by:
  - expr: ${{ item.priority }}
    direction: desc
  - expr: ${{ item.id }}
    direction: asc
```

未指定時は source array order。

Job priority は `order_by` より優先する。

## 22. Dynamic Job dependency

Dynamic Job も通常の `needs` として扱う。

```yaml
aggregate:
  needs: [evaluate]
```

この場合 `evaluate` は、その template から生成された Job 群全体を意味する。

後段 Job は生成 Job 群が必要条件を満たすまで待つ。

生成 Job の Output 集合は stable key 付き map として扱うことを基本とする。

特定 key の Job を個別参照する仕組みも許可する。

## 23. Reusable Workflow

MVP から対応する。

```text
Parent Workflow Run
└─ Job
   └─ Child Workflow Run
```

- Parent から Input を明示的に渡す
- Child Output を Parent Job Output として mapping 可能
- Child は独立 Workflow context / state を持つ
- Parent mutable state を Child から直接変更しない
- Artifact も明示的に受け渡す
- circular reference を検出して拒否
- Cancel は Child へ伝播
- Child failure は Parent Job へ伝播
- Parent Job Retry では新しい Child Workflow Run を作る

Reusable Workflow の reference depth に固定上限は必須としないが、cycle detection は必須。

## 24. External LLM executor

External LLM は pull-based executor とする。

理由:

Local MCP Server から ChatGPT 側へ任意タイミングで push 実行を要求できないため。

概念フロー:

```text
Job ready
↓
waiting_external
↓
task_claim
↓
lease issued
↓
external execution
↓
task_submit
↓
completed
```

### 24.1 Claim / Lease

`task_claim` は atomic。

同一 task の複数同時 claim は 1 件だけ成功。

Lease は少なくとも以下を持つ。

```text
task_id
lease_id
claimed_by
expires_at
```

MVP default lease duration: 60 分。

System / Workflow / Job で変更可能。

期限切れ後の古い lease からの `task_submit` は拒否する。

Lease expiry 後は policy により requeue / failure を選択可能にする。

### 24.2 Submit chaining

`task_submit` で結果を送る際、次に実行可能な External Job を同時に返せる optional behavior を持つ。

これにより ChatGPT / Agent が連続して処理しやすくなる。

## 25. Human Review

Human Review は `executor: human` で任意位置に置ける。

状態:

```text
waiting_review
```

MVP outcome:

- approve
- reject

Approve は Job success。

Reject は `human_rejected` failure。

Human Review に lease は設けない。

失敗 Job を人間操作だけで success に書き換えることは禁止する。

修正後は Retry する。

## 26. Workflow concurrency

論理的な Workflow Run concurrency を設定可能にする。

例:

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

`on-limit`:

- `queue`
- `reject`

未指定時は無制限。

Runner Pool の物理 Runner 数とは別概念。

## 27. Persistence

標準 backend は SQLite。

- JobRunner 専用 DB file を既定とする
- parent DB との同居を必須にしない
- persistence layer は interface 化し、将来 PostgreSQL 等へ交換可能にする

SQLite default:

- WAL
- `foreign_keys = ON`
- `busy_timeout` 設定

Migration は Core が番号付き SQL migration を管理する軽量方式を基本とする。

### 27.1 主な tables

概念上、少なくとも以下を持つ。

```text
workflow_runs
job_runs
job_attempts
job_steps
workflow_state
workflow_state_history
artifacts
events
execution_logs
runners
external_tasks
external_leases
human_reviews
idempotency_records
```

必要に応じて concurrency 管理 table を追加する。

exact column / index / constraint は詳細設計で確定する。

## 28. Event Log / Execution Log

### 28.1 Event Log

構造化された append-only audit log。

代表 event:

- workflow started / completed
- job queued / started / completed
- retry requested / started
- task claimed / submitted
- human review submitted
- runner lost / restarted / restart suppressed
- artifact registered
- pause / resume / cancel
- retention deletion

Actor / source も記録できるようにする。

### 28.2 Execution Log

Job / Attempt の実行ログ本文。

DB に大きな log text を保存せず file として保持する。

```text
runs/<workflow_run_id>/logs/<job_id>/<attempt>.log
```

DB には path / size / timestamps 等の metadata を保存する。

Step boundary を log 内で表現し、WebUI で折りたためる構造を維持する。

## 29. Progress

Action は明示的に progress を報告可能。

概念 API:

```text
progress(current, total, message)
```

Workflow 全体は Job completion から自動 progress を算出できる。

Reusable Workflow は Child progress を Parent Job progress へ集約可能にする。

## 30. Retention

Retention 機能は用意するが、既定は無期限保存。

対象:

- Workflow Run records
- Event metadata
- Execution Log
- Artifact metadata

Artifact 実体の削除は原則として親システム側の責任。

Core が管理対象を削除した場合、削除 Event を残す。

## 31. Authorization

認証は親システムの責任。

Core には将来 RBAC / ABAC / RLS 相当へ拡張できる authorization hook を最初から用意する。

概念型:

```text
ActorContext
AccessScope
AuthorizationProvider
```

MVP default:

```text
AllowAll
```

ただし Service operation は常に AuthorizationProvider を通す。

Adapter から repository / DB を直接変更する経路は作らない。

## 32. Secrets

Secrets は親システムが runtime 起動時に提供する。

YAML に secret value を直接保存しない。

例:

```yaml
with:
  token: ${{ secrets.API_TOKEN }}
```

Secrets は原則として以下へ平文保存しない。

- SQLite
- Event Log
- Execution Log
- Debug IPC log

親システム再起動後は親側から再注入する。

## 33. Idempotency

状態変更系 Service / MCP operation は request / idempotency key を optional で受け付ける。

主な対象:

- workflow start
- task submit
- review submit
- retry
- pause
- resume
- cancel

同じ key が再送された場合、同じ side effect を重複実行せず、最初の結果を返す。

分散 exactly-once guarantee までは提供しない。

## 34. Service API / MCP

WebUI / MCP / Python API は同じ Runtime Service を呼ぶ。

MCP public tool name は親システム namespace を必須とする。

例:

```text
novel_wf_start
novel_wf_info
novel_wf_task_claim
fx_wf_start
fx_wf_info
fx_wf_task_claim
```

主な logical operation:

```text
wf_start
wf_list
wf_info
wf_pause
wf_resume
wf_cancel
wf_retry
wf_task_info
wf_task_claim
wf_task_submit
wf_review_submit
wf_artifact_info
wf_log_read
wf_runner_info
```

Read API は必要に応じて以下の include option を持てる。

```text
include_jobs
include_attempts
include_steps
include_artifacts
```

大きい Execution Log 本文は separate read API とする。

親システムは不要な MCP tool を公開しない選択ができる。

## 35. Static validation

「実行してから気付く」より、開始前に厳格に拒否する。

Validation は少なくとも以下の 2 段階。

1. YAML load 時
2. Workflow Run start 時（runtime input / current registry を含む）

検証対象:

- YAML syntax
- schema
- duplicate Job ID
- `needs` target existence
- DAG cycle
- Action existence
- Action version
- Runner Pool existence
- Input / Output reference
- Input type
- CEL compile
- JMESPath compile
- Retry values
- timeout
- priority
- `foreach`
- dynamic key / max count
- Reusable Workflow existence
- Reusable Workflow cycle
- concurrency settings
- Output Schema

推測で補正せず、path と理由を含む明確な validation error を返す。

## 36. Reuse

自動的な Job result reuse は同一 Workflow Run 内に限定する。

別 Workflow Run / 別 Workflow Definition からの global cache は行わない。

Reuse 判定に必要な条件:

- final Job Input が同一
- upstream Artifact が同一
- Workflow Definition 全体が同一
- Action ID が同一
- Action version が同一

比較用に hash を使用してよいが、複雑な全環境 identity system は作らない。

Workflow YAML が変更された場合、以前の結果は自動 reuse しない。

親システムが過去 Artifact を明示 input として渡すことは許可する。

## 37. Scheduler / Notification / CLI

MVP には含めない。

### Scheduler

将来 Scheduler を追加する場合も、既存 `workflow_start` Service API を呼ぶだけの Adapter とする。

Windows Task Scheduler 等の外部 Scheduler から同 API を呼ぶ構成を許容する。

### Notification

Mail / Slack / Push 等は親システムが Event / API を利用して実装する。

### CLI

MVP では WebUI / MCP を主操作面とするため CLI は作らない。

## 38. 任意コード実行

Core は arbitrary code / shell executor を標準提供しない。

必要な親システムが専用 Action を登録する。

例:

```text
python_exec Action
shell_exec Action
container_exec Action
```

隔離が必要なら、その Action 側で Docker / Sandbox 等を利用する。

JobRunner Core はその実行方式を特別扱いしない。

## 39. WebUI

WebUI は後続設計とする。

方向性としては簡易 viewer ではなく、Human operator が通常運用できる管理画面をしっかり作る。

最低限必要になる機能候補:

- Workflow 一覧 / 詳細
- Workflow Run 一覧 / 詳細
- DAG / Job state
- Attempt / Step
- Execution Log
- Artifact metadata
- Human Review
- Retry
- Pause / Resume
- Cancel
- Workflow start
- Runner / Runner Pool 状態
- failure reason / Event history

画面構成・UX・技術スタックは別途決定する。

## 40. MVP default values

現時点の既定値を以下とする。

| 項目 | 既定値 |
| --- | --- |
| Dynamic Job 生成上限 | 1000 / Workflow Run |
| Dynamic Job 入れ子深さ | 固定上限なし |
| Heartbeat interval | 5 秒 |
| Runner lost 判定 | 20 秒 |
| External LLM lease | 60 分 |
| Job execution timeout | なし |
| Automatic retry | なし |
| Retention | 無期限 |
| AuthorizationProvider | AllowAll |
| Workflow concurrency | 無制限 |
| Runner assignment | pull |
| 同一 Workflow Run の internal Job 同時実行 | 最大 1 |

## 41. 実装原則

1. GitHub Actions に対応する概念は可能な限り同じ用語を使う。
2. Core は親システム固有の業務ロジックを持たない。
3. 既存 OSS / library で十分な機能は再発明しない。
4. MCP / Web は Adapter とし、Runtime Service を唯一の状態変更経路とする。
5. 永続化して復旧可能にするが、分散システム規模の exactly-once / consensus は目指さない。
6. Runner / Action failure を明確に区別する。
7. Workflow Run 内のデータ受け渡しは Input / Output / state / Artifact として明示する。
8. Runner local memory / temporary file の暗黙引き継ぎに依存しない。
9. 危険な任意コード実行や Sandbox は親側 Action の責任とする。
10. 不正・曖昧な Workflow Definition は fail-closed で拒否する。

## 42. 後続詳細設計

次の段階で、以下を exact contract として固定する。

- YAML JSON Schema / Pydantic model
- SQLite table / index / constraint
- Service API request / response
- MCP tool input / output
- JSON Lines IPC event schema
- CEL environment / helper list
- JMESPath integration contract
- Job / Workflow Run state transition table
- Retry / Recovery state machine
- Runner restart state machine
- External lease state machine
- Reusable Workflow resolution
- Dynamic Job expansion identity / dependency resolution
- Error code catalog
- Python package public API
- Test strategy
- WebUI detailed design
