# 03. Runtime / Scheduling 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/01-workflow-definition.md`
  - `docs/detailed-design/02-expression-and-inputs.md`
- 用語方針: GitHub Actions に対応概念がある場合は、可能な限り同じ用語を使う

## 1. 目的

本書は JobRunner における Runtime の起動、Workflow Run / Job の状態遷移、ready 判定、Scheduling、Runner への割当、Priority、Pause / Resume / Cancel、Workflow concurrency を定義する。

対象:

- Workflow Run 開始処理
- Job Run の生成
- Job の ready / blocked 判定
- `needs` と `if` の評価タイミング
- internal / external_llm / human executor の Scheduling
- Runner Pool 選択
- Runner pull
- atomic claim
- 1 Workflow Run 内 internal Job 同時実行最大 1
- Workflow Run / Job priority
- Dynamic Job の `order_by` との関係
- Workflow concurrency
- Pause / Resume
- Cancel
- Workflow Run の最終 conclusion
- Runtime 再起動時の基本 recovery entry point

Runner Process、Heartbeat、Action Runner Process の実装詳細は `04-runner-and-ipc.md`、Retry / Recovery の詳細は `10-retry-recovery-cancel.md` で定義する。

## 2. 基本原則

1. Runner が取得する実行単位は Job とする。
2. Runner は Workflow Run を占有しない。
3. 同一 Workflow Run では internal Job を同時に 2 件以上実行しない。
4. 別 Workflow Run は Runner Pool の空き数まで並列実行できる。
5. External LLM 待ち、Human Review 待ちでは Runner を保持しない。
6. Scheduling は DB 上の永続状態を Source of Truth とする。
7. Runner への Job 割当は atomic に行い、同一 Job を複数 Runner が取得できないようにする。
8. Priority 変更は未開始 Job の選択順だけに影響し、実行中 Job を preempt しない。
9. Pause は実行中 Job を止めず、新しい Job の開始だけを止める。
10. Cancel は graceful を標準とする。
11. Runtime / Runner 再起動後も DB を基に Scheduling を再開できる。

## 3. Runtime 起動

親システム起動時に JobRunner Runtime を初期化する。

概念順序:

1. JobRunner 設定をロード
2. Persistence を初期化
3. Migration を適用
4. Action Registry を bootstrap
5. Workflow Definition Registry / Loader を初期化
6. Runner Pool 定義を検証
7. Runtime instance を登録
8. 未完了 Workflow Run / Job / Runner 状態を recovery 対象として走査
9. Runner Pool を起動
10. Scheduling を受付可能状態にする

Runtime 起動に失敗した場合、Runner Pool は起動しない。

## 4. Runtime instance

親システム起動ごとに一意の `runtime_instance_id` を発行する。

用途:

- 現在有効な Runner 群の識別
- 親システム再起動前の孤児 Runner との区別
- recovery 判定
- Event / 診断情報

古い `runtime_instance_id` に属する Runner は、新しい Runtime から Job を claim できない。

## 5. Workflow Run 開始

### 5.1 start request

Workflow Run 開始 API は少なくとも以下を受け取る。

```text
workflow_id
input
priority (optional)
idempotency_key (optional)
actor_context
source_identity (optional)
```

Workflow Definition の解決方法は親側 Loader / Registry に委ねる。

### 5.2 開始前検証

Workflow Run を DB に作る前に、以下を検証する。

- Workflow Definition が存在
- YAML / typed model が有効
- Workflow Input が schema に適合
- 必須 Action が現在 Registry に存在
- Action version が解決可能
- `runs-on` の Runner Pool が存在
- `needs` DAG が有効
- CEL / JMESPath が compile 済み
- Reusable Workflow reference が有効
- Workflow concurrency expression が評価可能
- Dynamic Job 上限設定が有効
- Authorization を通過

検証失敗時は Workflow Run を開始しない。

### 5.3 concurrency 判定

Workflow 定義に concurrency がある場合、Workflow Run 作成前に `group` を評価する。

```yaml
concurrency:
  group: ${{ inputs.symbol }}
  max-runs: 1
  on-limit: queue
```

`group` の最終値は Workflow Run に snapshot する。

同じ group の active Workflow Run 数が `max-runs` に達している場合:

- `on-limit: queue`: Workflow Run を作成するが Scheduling 対象外の待機状態にする
- `on-limit: reject`: 開始要求を拒否し、Workflow Run を作成しない

未指定時は concurrency 制限なし。

### 5.4 Workflow Run 作成 transaction

Workflow Run 開始時は 1 transaction で少なくとも以下を保存する。

- Workflow Run row
- Definition Snapshot
- Input Snapshot
- Action ID / version snapshot
- source_identity
- concurrency group
- priority
- 初期 Job Run rows
- `workflow_started` Event

静的 Job はこの時点で Job Run row を生成する。

Dynamic Job の生成対象は展開可能になるまで template 状態として保持し、実際の生成は `05-dynamic-jobs.md` に従う。

## 6. Workflow Run の状態

Workflow Run は少なくとも以下の実行状態を持つ。

```text
queued
running
paused
completed
```

完了時 conclusion:

```text
success
failure
cancelled
```

`paused` は conclusion ではない。

Concurrency 上限待ちは `queued` のまま別の待機理由を持たせる。

推奨内部 field:

```text
status
conclusion
pause_requested
cancel_requested
wait_reason
```

## 7. Job Run の状態

Job Run の外部表示は上位設計に従う。

主な `status`:

```text
queued
running
waiting_external
waiting_review
paused
completed
```

主な `conclusion`:

```text
success
failure
cancelled
skipped
blocked
```

`blocked` は依存関係等により実行不能が確定した Job に使用する。

## 8. Job ready 判定

Job は以下をすべて満たす場合に Scheduling 候補となる。

1. Job が未完了
2. Workflow Run が cancel 済みでない
3. Workflow Run が pause 中でない
4. Workflow concurrency 待ちでない
5. Dynamic Job の展開待ちでない
6. `needs` が必要条件を満たす
7. Job `if` が評価可能
8. `if` の結果が true
9. executor 固有の開始条件を満たす
10. internal Job の場合、同一 Workflow Run に別の running internal Job がない

### 8.1 `needs` 完了判定

通常 Job の `needs` は依存 Job が terminal になるまで待つ。

Dynamic Job template を `needs` に指定した場合、その template から生成された Job 群全体を 1 dependency group として扱う。

Dependency group の詳細は `05-dynamic-jobs.md` に従う。

### 8.2 依存失敗

依存 Job が failure / cancelled / blocked になった場合、後段 Job は即座に blocked にするのではなく、まずその Job 自身の `if` を評価する。

例:

```yaml
cleanup:
  needs: [build]
  if: ${{ always() }}
```

`always()` なら依存失敗後も実行可能。

`if` が未指定の場合は既定の `success()` 相当とし、必要な dependency が成功していなければ Job を `blocked` とする。

### 8.3 `if: false`

Job の `if` が false の場合、実行せず:

```text
status = completed
conclusion = skipped
```

とする。

`skipped` は failure と同義ではない。

## 9. Scheduler の責務

Scheduler は独立 daemon service を必須としない。

Core Runtime Service 内に以下の責務を持つ Scheduling component を置く。

- ready Job 判定
- Workflow concurrency 解放判定
- Dynamic Job 展開トリガ
- external / human wait への遷移
- internal Job の claim 候補生成
- terminal Job 後の downstream activation
- Workflow Run conclusion 再計算

Runner は Scheduling policy 全体を知らず、`claim_next_job(pool, runner)` を呼ぶだけでよい。

## 10. Executor ごとの Scheduling

### 10.1 internal

`executor: internal` または省略時の既定。

Job は ready になると Runner pull 対象になる。

### 10.2 external_llm

Job が ready になったら Runner には渡さず:

```text
status = waiting_external
```

へ遷移し、External task を作成する。

その後は `task_claim` / lease / `task_submit` で進行する。

### 10.3 human

Job が ready になったら Runner には渡さず:

```text
status = waiting_review
```

へ遷移し、Human Review record を作成する。

## 11. Runner pull

Runner は自分の Runner Pool を指定して Core に 1 件の Job を要求する。

概念 API:

```text
claim_next_job(
  runtime_instance_id,
  runner_id,
  runner_pool
)
```

返り値:

- claim 成功: Job / Attempt 実行情報
- 候補なし: `none`
- Runner 無効: error

Runner は busy 中に追加 Job を claim しない。

## 12. Job 選択順

internal ready Job の選択順は以下とする。

1. Workflow Run priority: 高い方を先
2. Job priority: 高い方を先
3. Dynamic Job `order_by`: YAML で定義された順
4. source / generated order
5. ready_at が古い方を先
6. 最後の deterministic tie-break として Job Run ID を使用

これにより同条件でも選択順が不定にならないようにする。

### 12.1 priority の型

Workflow Run / Job priority は整数とする。

既定値:

```text
0
```

大きい値を高優先度とする。

範囲は implementation safety のため signed 32-bit integer に収まる値を推奨し、schema で制限してよい。

### 12.2 実行中 Job

priority が変更されても実行中 Job は中断・差し替えしない。

変更は次回 claim から反映する。

## 13. 1 Workflow Run 内 internal Job 同時実行最大 1

internal Job claim 時に必ず以下を確認する。

```text
same workflow_run_id に running internal Job が存在しない
```

この条件は application-side check だけにせず、claim transaction 内で競合に耐える形で実装する。

具体的な DB lock / unique constraint / transaction pattern は `08-persistence.md` で確定する。

Runner が Job 完了後に Workflow Run を保持する必要はない。

例:

```text
Workflow Run A
Job 1 -> Runner 1
External LLM wait -> Runner 解放
Job 2 -> Runner 3
```

を許可する。

## 14. Atomic claim

Runner claim は 1 transaction で少なくとも以下を行う。

1. Runner が現在有効で idle であることを検証
2. Pool が一致する ready internal Job を選択
3. Workflow Run が pause / cancel / concurrency wait でないことを再確認
4. 同一 Workflow Run に running internal Job がないことを再確認
5. Job の現在 state が claim 可能であることを再確認
6. Job を `running` へ遷移
7. 新しい Attempt を作成
8. `runner_id` / `runtime_instance_id` を Attempt に記録
9. Runner を busy に更新
10. `job_started` Event を保存
11. claim payload を確定

どこかで競合した場合は rollback し、別候補を再選択できる。

二重 claim は禁止する。

## 15. Ready activation

Job / Attempt が terminal になったとき、Runtime は同じ transaction または直後の確実な activation 処理で以下を再評価する。

- downstream `needs`
- downstream `if`
- Dynamic Job 展開可能性
- Reusable Workflow 親 Job
- Workflow Run conclusion
- concurrency slot 解放

activation 自体は idempotent にする。

同じ Job を複数回 ready 化しても重複 Job Run / task を作らない。

## 16. Pause

Workflow Run に対する Pause request は状態変更操作として扱う。

### 16.1 Pause request

Pause 時:

- `pause_requested = true`
- Workflow Run `status = paused`
- running internal Job は継続
- 新しい internal Job claim は不可
- 新しい External task claim は不可
- waiting_external は保持
- waiting_review は保持
- 既に claim 済み External task の submit は受理
- Human Review の approve / reject は受理
- Job 完了に伴う内部状態更新は継続

Pause 中に Workflow Run の terminal 条件が満たされた場合は `completed` へ遷移してよい。

### 16.2 Resume

Resume 時:

- terminal Workflow Run には適用不可
- `pause_requested = false`
- conclusion 未確定なら `running` または `queued` へ戻す
- Scheduling を再開
- ready Job を再評価

Resume は Job Input や Definition Snapshot を変更しない。

## 17. Cancel

Cancel の詳細な Action Process 停止手順は `10-retry-recovery-cancel.md` で定義する。

Scheduling 観点では Cancel request 後:

- `cancel_requested = true`
- 新しい internal Job を claim しない
- 新しい External task claim を許可しない
- queued Job は cancelled
- waiting_external / waiting_review Job は cancelled
- running internal Job には graceful cancel を要求
- late External submit は拒否
- 新しい Dynamic Job 展開は行わない

全 Job が terminal になった時点で Workflow Run conclusion を `cancelled` とする。

## 18. Workflow Run conclusion

Workflow Run の conclusion は Job terminal 状態から導出する。

### 18.1 success

以下を満たす場合:

- cancel request で終了していない
- required Job に unhandled failure がない
- required Job が success または定義上許可された skipped 状態
- 実行可能 Job が残っていない

### 18.2 failure

以下のいずれか:

- `continue-on-error` 等で許可されていない Job failure
- blocked Job により Workflow の必要経路が完了不能
- Dynamic Job 展開失敗
- Reusable Workflow の unhandled failure
- Engine-level fatal error

### 18.3 cancelled

Workflow Run cancel が要求され、その結果として終了した場合。

Job単体の cancelled があっても、Workflow定義上処理可能で後続が成立する特殊ケースは将来許可可能だが、MVPでは通常 Workflow cancel へ集約する。

## 19. `continue-on-error`

`continue-on-error: true` の Job が failure になった場合:

- Job 自体の conclusion は `failure` のまま保持
- Workflow Run 全体の failure 判定では許容 failure として扱う
- downstream `needs` 参照からは実際の conclusion を確認可能
- `success()` helper 上の扱いは `02-expression-and-inputs.md` の定義に従う

失敗を success に書き換えない。

## 20. Workflow concurrency slot 解放

Concurrency group の active count に含める Workflow Run は少なくとも以下:

```text
queued (concurrency wait 自身を除く active holder)
running
paused
```

`completed` は含めない。

slot が空いたら、同じ group で `on-limit: queue` の待機 Workflow Run を次の順で 1 件以上解放する。

1. Workflow Run priority
2. requested_at / created_at が古い順
3. Workflow Run ID

解放数は `max-runs` の空き数まで。

Pause 中 Workflow Run は slot を保持する。

## 21. Reusable Workflow と Scheduling

Reusable Workflow Job が ready になった場合、親 Job は子 Workflow Run を開始し、その完了を待つ。

親 Jobは internal Runner を占有し続けない。

概念状態:

```text
Parent Job
  -> child Workflow Run start
  -> waiting_child
  -> child completed
  -> Parent Job terminal
```

外部表示で `waiting_child` を独立 status として公開するかは Service API 詳細設計で決めるが、内部 wait reason として識別可能にする。

子 Workflow Run は通常の Scheduling / concurrency ルールに従う。

## 22. Dynamic Job と Scheduling

Dynamic Job template が展開可能になった時点で expansion を行う。

生成 Job は通常 Job と同じ Scheduling 対象となる。

同一 Workflow Run 内 internal Job 同時実行最大 1 の制約は生成 Jobにも適用する。

Dynamic Job `order_by` は Job priority より下、source order より上の並び順として扱う。

詳細は `05-dynamic-jobs.md`。

## 23. Runtime 再起動時の Scheduling recovery

Runtime 起動時に未完了 Workflow Run を走査する。

最低限:

- queued Job: ready 条件を再評価
- waiting_external: task / lease 状態を照合
- waiting_review: review state を照合
- running internal Job: Runner / Heartbeat / Attempt ownership を recovery 対象へ渡す
- paused Workflow Run: pause を維持
- concurrency wait: slot を再計算
- terminal Job: downstream activation 漏れがないか idempotent に再評価

running Job を即座に成功扱いしたり queued に戻したりせず、Runner recovery policy に従う。

## 24. Polling / wake-up

MVP では複雑な message broker を導入しない。

Runner は pull 型で、候補なしの場合は低負荷で待機する。

Runtime 内部は以下のイベントで Scheduler を wake できる設計とする。

- Workflow Run start
- Job terminal
- Resume
- Retry
- External submit
- Human Review submit
- concurrency slot 解放
- Dynamic Job expansion

実装は `threading.Event`、condition variable、短い polling 等の軽量方式でよい。

DB を高頻度 busy-loop しない。

## 25. Fairness

厳密な公平性保証は MVP では行わない。

Priority が高い Workflow Run が継続的に投入された場合、低 Priority が長く待つ可能性は許容する。

ただし同一 Priority 内では `ready_at` / queue 時刻を使い、古い Job を優先する。

将来 starvation 防止が必要になれば aging を追加可能とする。

## 26. Scheduling で行わないこと

MVP Scheduler は以下を行わない。

- CPU / RAM / GPU 使用量による自動配置
- OS-level resource reservation
- 複数machineへの分散 scheduling
- preemption
- Runner Pool間の自動 fallback
- 未登録 Runner Pool の代替選択
- Cron / calendar scheduling
- 任意 remote runner discovery

## 27. Event

Scheduling に関する代表 Event:

```text
workflow_started
workflow_concurrency_queued
workflow_concurrency_released
workflow_paused
workflow_resumed
workflow_cancel_requested
workflow_completed
job_ready
job_started
job_skipped
job_blocked
job_completed
external_task_created
human_review_requested
```

Event の exact schema は `09-artifacts-logs-state.md` または専用 Event 設計で確定する。

## 28. 競合時の基本方針

同時操作が発生した場合は DB transaction 時点の最新状態を優先する。

例:

- Runner claim と Cancel が競合
- Resume と Cancel が競合
- Job completion と Runner lost 判定が競合
- concurrency slot 解放と新規 Workflow Run start が競合

状態遷移ごとに expected current state を条件に更新し、条件不一致なら最新状態を再読込して処理を終了または再評価する。

last-write-wins で Job state を無条件上書きしない。

## 29. Idempotency

Scheduling activation / state propagation は繰り返し実行可能にする。

以下を満たすこと:

- 同じ downstream Job を重複生成しない
- 同じ External task を重複作成しない
- 同じ Human Review request を重複作成しない
- terminal Job を再度 running にしない
- completed Workflow Run を再度 active にしない

外部 API request の idempotency key は `11-service-api-and-mcp.md` で定義する。

## 30. 受入条件

最低限以下を自動テストする。

### Workflow Run start

- valid Workflow が start できる
- invalid Input では Run を作らない
- 未登録 Action / Runner Pool では start できない
- `on-limit: reject` が Run を作らない
- `on-limit: queue` が待機 Run を作る

### Scheduling

- `needs` 完了前は後段が開始されない
- `if: false` は skipped
- dependency failure + default condition は blocked
- dependency failure + `always()` は実行可能
- Workflow Run priority が Job選択に反映される
- Job priority が反映される
- 同条件は deterministic 順になる

### Runner claim

- 複数 Runner の同時 claim で同一 Job を二重取得しない
- 同一 Workflow Run の internal Job は同時に 2 件 claim されない
- 別 Workflow Run は別 Runner で並列 claim できる
- Runner Pool 不一致 Job は claim しない

### Pause / Resume / Cancel

- Pause後に新Jobを開始しない
- Pause前から running のJobは継続できる
- Resumeでready Jobが再開される
- Cancel後に新Jobを開始しない
- queued / waiting Jobがcancelされる

### Concurrency

- `max-runs` を超えて同 group が active にならない
- slot解放でqueue待ちRunが進む
- priority順でslotが解放される

### Recovery

- Runtime再起動後もqueued Jobを再評価できる
- paused状態を維持する
- running Jobは勝手にqueued/successへ変更しない

## 31. 未確定事項

以下は後続詳細設計で確定する。

- SQLite の exact locking / index / unique constraint
- Runner heartbeat / process recovery の exact algorithm
- Retry / backoff / timeout / runner_lost の状態遷移
- External lease expiry の exact scheduling
- Reusable Workflow の `waiting_child` 外部表現
- Service API の exact request / response schema
