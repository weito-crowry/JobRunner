# 10. Retry / Recovery / Cancel 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/03-runtime-and-scheduling.md`
  - `docs/detailed-design/04-runner-and-ipc.md`
  - `docs/detailed-design/07-external-and-human.md`
  - `docs/detailed-design/08-persistence.md`

## 1. 目的

本書は JobRunner における Retry、Failure分類、Runner lost、Runtime再起動、Pause / Resume、Cancel、Timeout、Recovery policyを定義する。

## 2. 基本原則

1. Retryは新Attemptを作る。
2. 失敗Attemptをqueuedへ戻して再利用しない。
3. Retry Inputは元Job Inputから変更しない。
4. automatic retryはJob定義で明示時のみ有効。
5. manual retryはService APIから行う。
6. Job timeoutは未指定時なし。
7. Cancelはgracefulを標準とする。
8. force killは通常Service APIに含めない。
9. Recoveryは履歴を破壊せずfail-closedで行う。

## 3. Structured Failure

最低限:

```text
category
code
message
retryable
details optional
occurred_at
```

category例:

```text
execution
validation
protocol
timeout
runner
external
human
cancel
internal
```

code例:

```text
action_exception
action_process_exit
output_validation_failed
ipc_protocol_error
job_timeout
runner_lost
action_version_mismatch
external_lease_expired
human_rejected
cancelled
internal_error
```

## 4. Retry YAML

GitHub Actions風の全体記法へ寄せるが、`retry`自体はJobRunner独自拡張。

```yaml
retry:
  max-attempts: 3
  if: ${{ failure.retryable }}
  backoff:
    initial-seconds: 5
    max-seconds: 60
    multiplier: 2.0
```

## 5. Retry既定値

`retry`未指定:

```text
automatic retry = disabled
max-attempts = 1
```

manual retryは別扱い。

## 6. Automatic Retry判定

Attempt failure後:

1. failure確定
2. Jobがcancel対象か確認
3. `max-attempts`残数確認
4. retry `if` 評価
5. backoff算出
6. 次Attemptをqueued可能状態として予約
7. Event記録

`if`評価失敗はretryしない。Jobをfailureとして確定し、式評価failureを記録する。

## 7. Backoff

MVPで以下を許可する。

```text
initial-seconds
max-seconds
multiplier
```

jitterは必須にしない。

0秒も許可。

backoff待ちはRunnerを占有しない。

## 8. Manual Retry

Service operation:

```text
wf_retry
```

対象をWorkflow Run / Job単位で指定可能にする設計余地を持つ。

MVP基本はfailed Jobを指定してRetry。

manual retryはauto retry上限消費後でも許可可能。

新Attempt番号を払い出す。

## 9. Retry Input固定

Retry時に以下を変更しない。

- Job final Input
- Workflow Definition snapshot
- Workflow Input snapshot

Action versionは元Workflow Run snapshot基準で照合。

current Registryが一致しなければRetry開始せず`action_version_mismatch`。

## 10. Retry Output / Artifact

失敗AttemptのOutputをcurrent成功Outputとして利用しない。

新Attempt success後、そのOutput/Artifactがcurrentになる。

旧Attempt成果物は履歴として残す。

## 11. Job Timeout

YAMLで明示時のみ有効。

```yaml
timeout-minutes: 120
```

またはJobRunner独自の細粒度記法を追加してもよいが、可能な限りGitHub Actions名へ寄せる。

未指定は無期限。

## 12. Timeout処理

期限到達時:

1. RunnerがAction Runnerへcancel request
2. graceful終了猶予
3. 終了しない場合はRunnerがAction子Processをterminate可能
4. Attemptをfailure / `job_timeout`で確定
5. Retry policy適用

Job timeoutとExternal Lease timeoutは別。

## 13. Runner Lost

Heartbeat期限超過でRunner lost判定。

対象Runnerが所有していたrunning Attemptを`runner_lost` failureとして扱う。

重複Recovery防止のため、conditional updateで未確定Attemptだけを確定する。

その後Retry policyを適用。

## 14. Runner Restart

Runner Process再起動policy:

```text
mode: on_failure | never
max_restarts
window_seconds
backoff
```

Crash loop判定時はrestart suppressed。

suppressed状態をService/MCP/Webから取得可能にする。

## 15. Parent Runtime再起動

新Runtime起動時:

1. 新`runtime_instance_id`発行
2. DB migration確認
3. Registry / Runner Pool構築
4. non-terminal Workflow Run列挙
5. running AttemptとRunner heartbeat照合
6. old runtime所属Runnerをcurrent ownershipとして認めない
7. orphan running AttemptをRecovery
8. External valid leaseは維持
9. waiting_reviewは維持
10. queued/ready判定再開

## 16. Old Runner fencing

旧Runtimeまたは旧Runner instanceからのlate updateを拒否する。

更新要求には最低限:

```text
runtime_instance_id
runner_id
runner_instance_id
attempt_id
```

を含め、current ownershipと一致する場合のみ受理。

## 17. Pause

Workflow Run単位。

Pause要求時:

- `pause_requested=true`
- running internal Jobは継続
- 新internal Jobをclaimさせない
- 新external task claimを許可しない
- waiting_reviewは保持
- claim済みexternal submitは受理
- review submitは受理

## 18. Resume

`pause_requested=false`にし、ready判定 / schedulingを再開。

Resume自体で新Attemptは作らない。

## 19. Cancel

Workflow Run cancel時:

- `cancel_requested=true`
- queued / ready Jobをcancelledへ
- waiting_external task/leaseを無効化
- waiting_reviewをcancelledへ
- running internal Actionへcancel request
- Reusable Workflow childへcancel伝播
- 新Job activation禁止

## 20. Graceful Cancel

Action Runtime Handleでcancel確認可能にする。

```python
if runtime.cancel_requested():
    return ...
```

協調停止を基本とする。

Actionが応答しない場合のterminateはRunner運用上の最後の手段であり、一般利用者向けforce-success/force-cancelとは分ける。

## 21. Cancel conclusion

cancel要求により終了したJob:

```text
status=completed
conclusion=cancelled
```

Workflow Runも必要Jobがcancelされた場合、最終conclusionはcancelled。

## 22. Human Reject

`human_rejected`はfailure。

人間操作だけでsuccessへ上書きしない。

必要ならJob Retry。

## 23. External Lease expiry

Policy:

```text
requeue
fail
```

`fail`時はstructured failureを生成しRetry policyへ渡す。

`requeue`時は同一Attemptを再claim可能に戻す。

## 24. Recovery idempotency

Recovery処理は何度呼ばれても二重新Attemptや二重Eventを増やさないよう、状態条件とunique constraintを使う。

例:

```text
attempt terminalなら再確定しない
retry child既存なら再作成しない
child Workflow Run既存なら再作成しない
```

## 25. Workflow Run completion再評価

Recovery後、全Job状態を再評価しWorkflow Runがterminalになれる場合は確定する。

terminal Workflow Runを再openしない。

## 26. Failure propagation

通常はrequired Job failureでWorkflow failure。

`continue-on-error: true` Jobはdownstream条件評価に情報を残すが、Workflow最終failureへの寄与を抑制可能。

`if: always()`等は通常condition規則に従う。

## 27. Retry Event

代表:

```text
retry_scheduled
retry_started
retry_exhausted
manual_retry_requested
runner_lost
runner_restart_suppressed
recovery_started
recovery_completed
cancel_requested
cancel_propagated
```

## 28. 受入条件

1. auto retry disabled既定
2. max-attempts
3. retry if true/false
4. exponential backoff
5. manual retry
6. Retry Input固定
7. Action version mismatch拒否
8. timeoutなし長時間Job
9. timeout failure + retry
10. runner_lost recovery
11. runtime restart recovery
12. old runner late update拒否
13. pause中no new claim
14. resume
15. cancel queued/waiting/running
16. external late submit拒否
17. child workflow cancel伝播
18. recovery二重実行安全
19. terminal Runを再openしない
