# 03. Runtime / Scheduling 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `05`, `08`, `10`

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
10. External Lease/Retry backoff等の期限処理はRuntime内部Maintenance Loopが担当する。

## 2. Runtime起動

1. Config/data root/Storage解決
2. Migration
3. Action Registry / Validator Registry bootstrap
4. Workflow Resolver
5. Runner Pool validation
6. runtime_instance_id発行
7. non-terminal Recovery
8. Runner Supervisor
9. Maintenance Loop
10. Scheduling開始

## 3. Workflow Run start

Start前にDefinition/Input/Action+Validator version/Runner Pool/Expression/Reusable/Concurrency/Authorizationを検証。

Concurrency groupは`01`に従いnon-empty string。暗黙trim/lowercase無し、case-sensitive完全一致。

Start transaction:

- Run snapshot
- Action/Validator version snapshot
- concurrency snapshot
- static Job/Dynamic template metadata
- Event
- idempotency result

Concurrency `queue|reject` は`08`のatomic holder countで処理。

## 4. Status

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

## 5. Job activation

条件:

1. non-terminal
2. Run cancel/pause/concurrency waitでない
3. Dynamic parent/global dependency準備済み
4. condition dependency set terminal
5. `if`評価

`if` helper/false semanticsは`02`。

Executor activation:

- internal: final Input/continue-on-errorをsnapshotしqueued Runner対象。Attemptはclaim時作成
- external: activation時Attempt + Task、waiting_external
- human: Attempt + Review、waiting_review
- reusable: binding + Attempt + Child、waiting_child

Retry予約だけではAttemptを作らない。

## 6. Internal / External selection order

共通ordering軸:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source/generated order ASC
5. ready_at ASC
6. Job Run ID

Priorityは`01`のsigned 64-bit integer。InternalはPool適合も条件。External claimも同順序。

## 7. Atomic internal claim

1 transaction:

1. Runner current/idle/pool
2. candidate/current Run gate
3. same Run running internal無し
4. Job queued + retry_not_before到達
5. Job running
6. new Attempt
7. ownership
8. Event

DB partial uniqueでもone-runningを保証。

## 8. Maintenance Loop

Maintenance LoopはWorkflow Scheduler/Cronではなく、既存Runtime状態の**期限到来処理専用**の親Process内thread/task。

対象:

```text
retry_not_before
external lease expires_at
concurrency wait再確認 wake-up
```

Runner heartbeat/livenessはRunner Supervisorが担当し、Job timeoutはRunner自身が担当する。

### 8.1 wait方式

Busy loopしない。

1. DB/Memory状態から最も近いdeadlineを求める
2. condition/eventで待つ
3. 新deadline追加時はnotify
4. deadline到達または最大sleep到達でdue itemを再query

System default:

```text
maintenance_max_sleep_seconds = 5
```

finite positiveで設定変更可。

### 8.2 retry deadline

`retry_not_before <= now` のJobを新Attempt作成せずRunner/activation対象としてwakeする。

Pause中はdeadline到来を記録してもnew activationしない。Resumeで進める。

### 8.3 external lease deadline

`expires_at <= now` のactive Leaseを`07`のexpiry policyでconditional transaction処理する。

Pause中もLease clockは止めず、expiry policyは実行する。

Runtime restart時は起動Recoveryで既に期限超過の項目を先に処理し、その後Maintenance Loopへ移行する。

## 9. Terminal後 activation

Terminal transition後idempotentに:

- downstream needs/if
- Dynamic expansion
- Reusable parent
- Result Reuse pending check
- Workflow Output/conclusion
- concurrency release

を再評価し、Maintenance Loopへ必要なdeadline/wake eventをnotifyする。

## 10. Result Reuseの目的

Parent restart/Resume/Manual Retry後に、既に成功したJobを再実行せず使ってよいかを厳格に判定するための機能。Global cacheではない。

別Workflow Runの結果は自動reuseしない。過去Artifactを親が明示Inputとして渡すことは別扱い。

## 11. Reuse Context / Key

Job execution開始時にnon-secret `reuse_context` を作る。

最低限:

```text
workflow_run_id
job_key
definition_hash
persistent_job_input_digest
direct_upstream_artifacts
executor_identity
validator_identity
```

`direct_upstream_artifacts`:

- condition dependency setの各dependencyについてcurrent Artifact identityをname順に収集
- identityは `artifact_id` + optional digest
- Artifact無しなら空集合

`executor_identity`:

- internal: `action_id + action_version`
- external: `jobrunner.external_llm.v1`
- human: `jobrunner.human.v1`
- reusable: `reusable_binding.child_definition_hash + child_action_versions + child_validator_versions`

`validator_identity`:

- validator無し: null
- validator有り: `validator_id + validator_version`

Canonical JSONをSHA-256し `reuse_key` とする。用語として「fingerprint」は使わない。

## 12. Reuse eligibility

Successful Attemptは原則reuse eligible。ただし:

- ActionがRuntime Handle `state.get` を実行
- ActionがJob Input/direct upstream Artifact以外のArtifactをdynamic materialize
- 親Adapterが明示 `reuse_eligible=false` としたexecutor extension

ならfalse。

ValidatorはJob定義とversion snapshotへ含まれるため、Validator使用だけではreuse不適格にしない。

## 13. Reuse判定条件

同一Workflow Run内でのみ:

1. final persistent Job Input
2. direct upstream Artifact identities
3. entire Workflow Definition hash
4. executor identity
5. validator identity
6. stored result Payload integrity
7. current Registryがsnapshot Action/Validator versionを提供可能
8. `reuse_eligible=true`

が全て一致する場合のみreuse。

Secret materialized valueは比較しない。Secret rotation互換性は親側Action/Validator/Workflow version管理責任。

## 14. Manual Retry後のdescendant処理

### blocked / skipped descendants

Target dependency closureのblocked/skippedをactivation再評価。

### successful descendants

`reuse_check_pending=true`。

Dependencies terminal後:

- match -> success維持、`job_result_reused`
- mismatch/ineligible/payload missing/version unavailable -> `successful_job_result_not_reusable`

MVPではsuccess Jobをnew Inputで同じJob Run内に自動再実行せずnew Workflow Runを要求。

### failed descendants

Target以外を勝手にRetryしない。

## 15. Pause / Resume

Pause:

- Run paused
- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human review/started Child進行可
- Maintenance LoopのLease expiryは継続

Resumeでactivation再評価。new Attemptは作らない。

## 16. Cancel

- cancel_requested
- queued/waitingをcancel
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
- Dynamic expansion failure
- Child failure
- Workflow Output failure
- successful_job_result_not_reusable
- engine fatal

Cancel request由来はcancelled。

## 18. Concurrency / Recovery

Concurrency holder/releaseは`08`。Group比較case-sensitive BINARY semantics。

Runtime restart:

- queued activation再評価
- due retry/external Leaseを即時処理
- Review/Child関係復元
- running internal -> `10` Runner recovery
- paused維持
- completed success Job保持
- pending reuse check再開

Recoveryだけでcompleted Runをreopenしない。

## 19. 受入条件

1. internal one-running
2. deterministic ordering
3. concurrency group identity
4. Maintenance Loop no-busy-loop
5. retry deadline wake <= max sleep lag
6. external Lease expiry without external traffic
7. paused Lease expiry processing
8. restart overdue timer processing
9. same-run reuse only
10. reuse key includes Action+Validator version
11. state.get/dynamic artifact ineligible
12. Manual Retry descendant reuse
13. no cross-Run reuse
