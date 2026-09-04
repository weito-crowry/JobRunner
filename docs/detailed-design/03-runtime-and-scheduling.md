# 03. Runtime / Scheduling 詳細設計

- Status: Draft v1.0
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `05`, `08`, `09`, `10`

## 1. 基本原則

1. Runnerが取得する単位はJob。
2. RunnerはWorkflow Runを占有しない。
3. 同一Workflow Run internal Job同時実行最大1。
4. 別Workflow RunはRunner Pool空きまで並列。
5. External/Human/Reusable待ちはRunner非占有。
6. DB永続状態がSource of Truth。
7. claim/state transitionはconditional transaction。
8. Priority変更はpreemptしない。
9. Job result自動再利用は同一Workflow Run内だけ。別Run/global cache無し。
10. External Lease/Retry backoff/Retention等の期限処理はRuntime内部Maintenance Loopが担当する。

## 2. Runtime起動

Parent Processの正規起動順:

1. Core config / data root / Integration Bootstrap entrypoint解決
2. `Integration Bootstrap(role=parent)`
3. Registry/Auth/Secrets/optional Store factory構築
4. SQLite / PayloadStore / ArtifactStore初期化
5. Migration
6. Workflow Resolver
7. Runner Pool config validation
8. `runtime_instance_id`発行
9. non-terminal Recovery
10. Runner Pool Supervisor起動
11. Maintenance Loop起動
12. Scheduling受付開始

Bootstrap/Provider/Storage/Migration/Pool validation失敗時はScheduling開始しない。

## 3. Workflow Run start

Start前にDefinition/Input/Action+Validator version/Runner Pool/Expression/Reusable/Concurrency/Authorizationを検証。

Start transaction:

- Run Definition/Input/effective settings/Retention snapshot
- Action/Validator version snapshot
- concurrency snapshot
- static Job/Dynamic template rows
- Event
- optional idempotency result

Static/generated Job rowは最初 `status=queued` でも、**Runner取得可能とは限らない**。`ready_at=NULL` はdependency/condition activation未完了を意味する。

Concurrency `queue|reject` は`08`。

## 4. Status / readiness

Workflow:

```text
queued|running|paused|completed
```

Conclusion:

```text
success|failure|cancelled
```

Job:

```text
queued|running|waiting_external|waiting_review|waiting_child|completed
```

Conclusion:

```text
success|failure|cancelled|skipped|blocked
```

Jobに`paused` status無し。

`queued` Job:

- `ready_at=NULL`: dependency/condition activation待ち
- `ready_at!=NULL`: executor開始可能なsnapshot確定済み
- `retry_not_before>now`: Retry backoff待ち

Internal Runner claimには `ready_at IS NOT NULL` が必須。

## 5. Job activation

Activation前提:

1. Job non-terminal
2. Run cancel/pause/concurrency waitでない
3. Dynamic parent/global dependency準備済み
4. condition dependency set terminal
5. Job `if`を現在contextで評価

### 5.1 `if=false`

Attemptを作らずJob=`completed/skipped`。Dynamic templateは`05`のexpansion outcomeを使う。

### 5.2 `if=true`

1. `continue-on-error`評価/snapshot
2. `with`を解決し`02`の `persistent_input + secret_bindings + input_digest` 作成
3. executor別activation

Internal:

- Job rowへ `pending_input_json/pending_secret_bindings_json/pending_input_digest` を保存
- `ready_at=now`
- status=`queued`
- Attemptはまだ作らない

External:

- AttemptへInput snapshot保存
- Task作成
- status=`waiting_external`

Human:

- AttemptへInput snapshot保存
- Review作成
- status=`waiting_review`

Reusable:

- binding確定
- AttemptへInput snapshot保存
- Child作成
- status=`waiting_child`

Input/condition resolution failureがAttempt作成前ならJob-level failureで、Retry Inputが無いため`10`のManual Retry対象外。

## 6. Internal / External selection order

共通ordering軸:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source/generated order ASC
5. ready_at ASC
6. Job Run ID

InternalはPool適合も条件。External claimも同ordering。

## 7. Atomic internal claim

1 transaction:

1. Runner current/idle/pool
2. candidate/current Run gate
3. same Run running internal無し
4. Job `queued`
5. `ready_at IS NOT NULL`
6. `retry_not_before IS NULL OR <= now`
7. pending Input/bindings/digest存在
8. Job running
9. new Attemptへpending snapshot exact copy
10. Runner ownership
11. Event/Execution Log activation

Claim後pending snapshotはJob rowからclearしてよい。AttemptがSource of Truthになる。

DB partial uniqueでもone-runningを保証。

## 8. Maintenance Loop

Maintenance LoopはWorkflow Scheduler/Cronではなく期限到来・housekeeping専用。

対象:

```text
retry_not_before
external lease expires_at
concurrency wait wake-up
retention due items
orphan temp/payload/artifact cleanup
expired idempotency cleanup
```

Runner heartbeat/livenessはRunner Supervisor、Job timeoutはRunner自身。

### 8.1 wait方式

Busy loop禁止。Nearest deadline + condition/event wait。New earlier deadlineでnotify。

System default:

```text
maintenance_max_sleep_seconds = 5
```

finite positiveで変更可。

### 8.2 Retry deadline

`retry_not_before<=now` をactivation/claim対象としてwakeする。Pause中new Attempt開始無し。

### 8.3 External Lease deadline

active Lease `expires_at<=now` を`07` policyでconditional transaction処理。Pause中もclock継続。

### 8.4 Retention / cleanup

`01/08/09`のRun snapshot policyに従う。

- all unlimitedなら期限削除無し
- non-terminal current dataを期限だけで削除しない
- External Artifact実体delete無し
- FK依存順
- system retention audit Eventは通常event retention外
- idempotent

### 8.5 Restart

Recoveryでoverdue Retry/Lease、consistency cleanup、due Retentionをcurrent state条件付きで処理してからMaintenance Loopへ。

## 9. Terminal後 activation

Terminal transition後idempotentに:

- downstream needs/if
- Dynamic expansion/group propagation
- Reusable parent
- Result Reuse pending check
- Workflow Output/conclusion
- concurrency release
- Workflow progress再計算

を処理する。

## 10. Result Reuseの目的

Parent restart/Resume/Manual Retry後に、既に成功したJob結果を再実行せず使ってよいかを厳格に判定する。

Global cacheではない。別Workflow Run結果は自動reuseしない。過去ArtifactRefを明示Inputとして渡すことは別扱い。

## 11. Stored Reuse Context

Successful Attempt terminal時に最低限:

```text
workflow_run_id
job_key
definition_hash
input_digest
direct_upstream_artifacts
executor_identity
validator_identity
reuse_eligible
```

を保存する。

`direct_upstream_artifacts` はcondition dependency set各dependencyのcurrent Artifact identityをname順に収集する。Identity=`artifact_id + optional digest`。

Executor identity:

- internal: `action_id + action_version`
- external: `jobrunner.external_llm.v1`
- human: `jobrunner.human.v1`
- reusable: Child Definition hash + Child Action/Validator versions

Validator identity=無しnull、あり`validator_id+validator_version`。

Canonical JSON SHA-256をstored `reuse_key`にできる。

## 12. Reuse eligibility

Successful Attemptは原則eligible。ただし:

- Action Runtime Handle `state.get`
- persistent Input/direct upstream Artifact以外のArtifact materialize
- executor extensionが明示`reuse_eligible=false`

ならfalse。

Expressionの`state`利用は、それが`with`ならinput_digestへ入り、`if`なら§13でcurrent再評価されるため、それだけでineligibleにはしない。

## 13. Manual Retry後のsuccessful Job reuse check

**保存済みInput同士を比較するだけでは不可。** Current dependenciesから「今このJobを初めてactivationするならどうなるか」を副作用無しで再計算する。

順序:

1. dependency setがcurrent terminalになるまで待つ
2. current contextでJob `if`再評価
3. `if`がtrueであることを要求
4. current contextで`with`を再解決し、Secret valueはmaterializeせずexpected `persistent_input + secret_bindings + input_digest`を作る
5. expected input_digest == stored successful Attempt input_digest
6. current direct upstream Artifact identities == stored context
7. Definition hash/executor identity/Validator identity一致
8. current Registryがsnapshot version提供
9. stored Payload/required Managed Artifact integrity/availability確認
10. `reuse_eligible=true`
11. stored Outputをoptional JSON Schema -> Validator -> success_if -> SecretGuardで再検証

全て成功した場合のみsuccessを維持し `job_result_reused` Event。

以下はreuse不可:

- current `if=false`
- `if/with` expression error
- expected Input変更
- dependency Artifact変更/削除
- version不一致
- Payload破損/欠落
- Validator/Schema/success_if再検証失敗
- ineligible

Reuse不可時は既存success Jobを新Inputで同一Run内自動再実行しない。Workflow Runを `successful_job_result_not_reusable` / failure としnew Workflow Runを要求する。Reasonはdetailsへ `condition_changed|input_changed|artifact_changed|validation_changed|storage_unavailable|version_unavailable|ineligible` 等を入れる。

Secret materialized valueは比較しない。Secret rotation互換性は親version管理責任。

## 14. Dynamic expansion reuse

Manual Retry後、completed/success Dynamic template groupはGenerated Job reuseより先に`05`の**expansion reuse check**を通す。

- expansion set/source/key/order/item identityがexact match -> existing expansionを保持
- mismatch/error -> `dynamic_expansion_not_reusable`, retryable=false, new Workflow Run要求

Completed/skipped/blocked Dynamic templateは`10`のdescendant reactivation規則に従い、実行済みsuccess groupを暗黙再展開しない。

## 15. Pause / Resume

Pause:

- Run paused
- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human review/started Child進行可
- Lease expiry/Retention housekeeping継続

Resumeでactivation再評価。**既にready_at付きinternal JobのInput snapshotは再評価しない。**

## 16. Cancel

- cancel_requested
- queued/waiting cancel
- External Lease invalidation
- running internal cancel request
- Child cancel propagation
- new activation禁止

`always()`でもcancel後new Job開始無し。

## 17. Workflow conclusion

Success:

- required success
- intentionally skipped
- failure + continue-on-error
- reuse_check_pending無し

Failure:

- non-allowed failure/blocked
- Dynamic expansion/group failure
- `dynamic_expansion_not_reusable`
- Child failure
- Workflow Output failure
- `successful_job_result_not_reusable`
- engine fatal

Cancel request由来はcancelled。

## 18. Concurrency / Recovery

Concurrency holder/releaseは`08`。Group BINARY case-sensitive。

Runtime restart:

- Integration Bootstrap再構築
- queued Jobの`ready_at`/pending Input snapshotを復元
- unactivated queued Jobだけactivation再評価
- due retry/Lease
- Review/Child復元
- running internal -> Runner recovery
- paused維持
- success Job保持/reuse pending再開
- Dynamic expansion recovery
- Retention/housekeeping

Recoveryだけでcompleted Runをreopenしない。

## 19. 受入条件

1. Runtime startup/provider/storage order
2. static queued `ready_at=NULL` is not claimable
3. internal activation persists pending Input before claim
4. restart/resume keeps pending snapshot unchanged
5. claim copies pending snapshot exactly to Attempt
6. one-running/Pool ordering
7. Maintenance deadline/Retention behavior
8. same-run reuse only
9. successful reuse re-evaluates current `if`
10. successful reuse re-resolves expected Input/bindings
11. Artifact/version/Payload validation
12. stored Output Schema/Validator/success_if revalidation
13. no automatic changed-Input rerun
14. Dynamic expansion reuse gate
15. state.get/non-input Artifact ineligible
16. concurrency/recovery idempotency
