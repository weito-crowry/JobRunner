# 03. Runtime / Scheduling 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01-workflow-definition.md`, `02-expression-and-inputs.md`

## 1. 目的

Workflow Run開始、Job activation、Scheduling、Priority、Runner claim、Pause/Resume/Cancel、Concurrency、Workflow conclusionの正規規則を定義する。

## 2. 基本原則

1. Runnerが取得する単位は Job。
2. RunnerはWorkflow Runを占有しない。
3. 同一Workflow Runではinternal Job同時実行最大1。
4. 別Workflow RunはRunner Pool空き数まで並列可能。
5. External/Human/Reusable待ちではRunnerを保持しない。
6. DBの永続状態がSource of Truth。
7. claim/state transitionはconditional transactionで二重適用を防ぐ。
8. Priority変更は次回選択から反映しpreemptしない。

## 3. Runtime起動

順序:

1. Config / data root / DBを解決
2. Migration
3. Action Registry bootstrap
4. Workflow Resolver/Registry初期化
5. Runner Pool設定検証
6. `runtime_instance_id`発行
7. non-terminal Run/Job/Runner recovery
8. Runner Pool Supervisor起動
9. Scheduling受付開始

Runnerは別Processだが、必須の別HTTP/Broker serviceは置かない。Runner/Runtime間は`04`のinternal Service/Repository境界を使う。

## 4. Workflow Run start

request:

```text
workflow_ref
inputs
priority optional
source_identity optional
request_id optional
actor_context
```

開始前にDefinition/Input/Action version/Runner Pool/CEL/JMESPath/Reusable reference/concurrency/Authorizationを検証する。

### 4.1 concurrency

`group`をstart前に評価。active holder数が`max-runs`以上なら:

- `queue`: Run rowを作るが `wait_reason=concurrency` でScheduling対象外
- `reject`: Run rowを作らず拒否

### 4.2 start transaction

同一transactionで:

- Workflow Run row + definition/input/action version snapshot
- concurrency snapshot
- static Job rows / Dynamic template metadata
- `workflow_started` Event
- idempotency result（request_idがある場合）

## 5. Workflow Run status

```text
queued
running
paused
completed
```

conclusion:

```text
success
failure
cancelled
```

補助field:

```text
wait_reason nullable
pause_requested
cancel_requested
run_attempt >= 1
```

`run_attempt`は明示manual retryで完成済みfailed Runを再開した回数を含む論理Run試行番号。通常start時1。

## 6. Job status

MVP Job status:

```text
queued
running
waiting_external
waiting_review
waiting_child
completed
```

conclusion:

```text
success
failure
cancelled
skipped
blocked
```

**Job status に `paused` は持たない。** PauseはWorkflow Run側のScheduling gateであり、queued/waiting Jobの状態を変えない。

Dynamic template/expansion管理状態はJob statusと分離する。

## 7. Job activation

Jobをactivationできる条件:

1. terminalでない
2. Workflow Runがcancel中でない
3. Workflow Runがpause中でない
4. concurrency waitでない
5. Dynamic templateなら必要なparent/global dependencyが揃う
6. declared `needs` がterminal
7. `if`を評価可能

`if` helperとfalse時conclusionは`02`に従う。

- default `success()` false due non-allowed failure/blocked -> `completed/blocked`
- default false due skipped only -> `completed/skipped`
- explicit `if=false` -> `completed/skipped`
- cancel -> `completed/cancelled`

## 8. executor別 activation

### internal

final Input / continue-on-error snapshotを確定後、`queued`でRunner pull対象。

### external_llm

final Input確定後、Attemptとexternal taskを作り `waiting_external`。

### human

final Input確定後、Attemptとhuman review rowを作り `waiting_review`。

### Reusable Workflow

binding/Input確定後、Attempt + Child Workflow Runを作り親Jobを`waiting_child`。詳細は`06`。

Attempt作成時点はexecutorごとに上記またはRunner claim時。automatic Retry予約だけではAttempt rowを作らない。

## 9. Runner pull / Job選択

`claim_next_job(runtime_instance_id, runner_id, runner_instance_id, pool)`。

選択順:

1. Workflow Run priority desc
2. Job priority desc
3. Dynamic `order_by` rank
4. source/generated order
5. `ready_at` asc
6. Job Run ID deterministic tie-break

internal Job `runs-on`省略はSystem `default_runner_pool`へ解決済みであること。

## 10. Atomic internal claim

1 transactionで:

1. Runner current/idle/pool一致
2. candidate再確認
3. Workflow Run pause/cancel/concurrency再確認
4. 同じWorkflow Runにrunning internal Job無し
5. Jobがqueuedかつ`retry_not_before <= now`またはnull
6. Job -> running
7. **新Attempt row作成**
8. Runner ownership記録
9. Runner -> running/busy
10. Event + idempotent activation marker更新

SQLiteは`08`で定義する partial unique indexにより「1 Run internal running最大1」をDBでも防ぐ。

## 11. terminal後 activation

Job/Attempt/Child/External/Humanがterminalになるたびidempotentに:

- downstream needs/if
- nested/root Dynamic expansion
- Workflow output候補
- Workflow conclusion
- concurrency slot release
- reusable parent propagation

を再評価する。

## 12. Pause / Resume

Pause:

- `pause_requested=true`, Run status=`paused`
- running internalは継続
- 新internal claim不可
- 新external claim不可
- waiting_external/Review/Childは保持
- claim済みexternal submit、Human review submit、開始済みChildの進行は受理

Resume:

- `pause_requested=false`
- Runをqueued/runningへ戻しactivation再評価
- 新Attemptは作らない

同じ操作の再送はrequestが同等ならidempotent no-op。

## 13. Cancel

Workflow cancel:

- `cancel_requested=true`
- queued/waiting_external/waiting_review/waiting_childをcancel処理
- external lease invalidation
- running internalへcancel request
-開始済みChildへcancel propagation
- 以後新Job activation禁止

`always()`でもcancel後に新Job開始はしない。

## 14. Workflow conclusion

### success

全required Jobが:

- success、または
- intentionally skipped、または
- failureだがsnapshotted `continue-on-error=true`

であり、required blocked/cancelledが無い。

### failure

- non-allowed failure
- required blocked
- Dynamic expansion failure
- unhandled Child failure
- Workflow output evaluation failure
- engine fatal failure

### cancelled

Workflow cancel requestにより終了した場合。

## 15. `continue-on-error`

Job自身はfailureのまま。Workflow conclusionとdependencyの`success()`ではeffective successとして扱う。評価時点/Contextは`02`。

## 16. Workflow concurrency slot

active holder:

- queued/running/pausedで`wait_reason != concurrency`

concurrency待ちRunはholderに含めない。completedは含めない。Pause中はslotを保持。

解放順:

1. Workflow Run priority desc
2. created_at asc
3. Workflow Run ID

空きslot数まで解放する。start/slot release競合はtransactionでactive countを再確認する。

## 17. Runtime再起動

- queued: activation再評価
- waiting_external: task/lease照合
- waiting_review: review row照合
- waiting_child: child relation照合
- running internal: Runner ownership/heartbeatを`10`へ渡す
- paused:維持
- concurrency wait:slot再計算
- terminal: downstream activation漏れのみidempotent確認

Recoveryだけでcompleted Runを再openしない。completed/failed Runを再openできるのは明示manual retryのみ（`10`）。

## 18. wake-up

Runnerはcandidate無しならbounded waitする。Runtime側のEvent/condition/pollingで以下をwake triggerにする:

- Job terminal
- Resume/Retry
- External/Human submit
- concurrency release
- Dynamic expansion
- Child completion

DB busy-loopは禁止。

## 19. 競合規則

Cancel vs claim、completion vs runner_lost、start vs concurrency release等は「expected current state」をWHERE条件にしたconditional updateで解決する。更新0件なら最新状態を再読込し、古い操作で上書きしない。

## 20. 受入条件

1. start atomicity/idempotency
2. Workflow/Job status enum（Job paused不存在）
3. default/explicit Runner Pool routing
4. dependency success/failure/skipped/blocked helpers
5. internal 1 Run max1
6. multiple Run parallel
7. priority/order deterministic
8. external/human/reusable activation時Attempt作成
9. automatic retry予約時Attempt未作成
10. pause/resume
11. cancel no-new-activation
12. concurrency queue/reject/release race
13. restart recovery
14. recoveryがterminal Runをreopenしない
