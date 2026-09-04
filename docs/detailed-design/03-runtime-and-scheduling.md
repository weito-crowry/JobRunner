# 03. Runtime / Scheduling 詳細設計

- Status: Draft v0.3
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
9. **Job result自動再利用は同一Workflow Run内だけ。別Run/global cache無し。**

## 2. Runtime起動

1. Config/data root/Storage解決
2. Migration
3. Action Registry bootstrap
4. Workflow Resolver
5. Runner Pool validation
6. runtime_instance_id発行
7. non-terminal Recovery
8. Runner Supervisor
9. Scheduling開始

## 3. Workflow Run start

Start前にDefinition/Input/Action version/Runner Pool/Expression/Reusable/Concurrency/Authorizationを検証。

Start transaction:

- Run snapshot
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

InternalはPool適合も条件。External claimも同順序。

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

## 8. Terminal後 activation

Terminal transition後idempotentに:

- downstream needs/if
- Dynamic expansion
- Reusable parent
- Result Reuse pending check
- Workflow Output/conclusion
- concurrency release

を再評価。

## 9. Result Reuseの目的

Parent restart/Resume/Manual Retry後に、**既に成功したJobを再実行せず使ってよいか**を厳格に判定するための機能。

Global cacheではない。

別Workflow Runの結果は自動reuseしない。過去Artifactを親が明示Inputとして渡すことは別扱い。

## 10. Reuse Context / Key

Job execution開始時にnon-secret `reuse_context` を作る。

最低限:

```text
workflow_run_id
job_key
definition_hash
persistent_job_input_digest
direct_upstream_artifacts
executor_identity
```

`direct_upstream_artifacts`:

- condition dependency setの各dependencyについてcurrent Artifact identityをname順に収集
- identityは `artifact_id` + optional digest
- Artifact無しなら空集合

`executor_identity`:

- internal: `action_id + action_version`
- external: `jobrunner.external_llm.v1`
- human: `jobrunner.human.v1`
- reusable: `reusable_binding.child_definition_hash + child_action_versions`

Canonical JSONをSHA-256し `reuse_key` とする。用語として「fingerprint」は使わない。

## 11. Reuse eligibility

Successful Attemptは原則reuse eligible。ただし以下の場合はautomatic reuse checkで証明不能として`reuse_eligible=false`:

- ActionがRuntime Handle `state.get` を実行した
- ActionがJob Input/direct upstream Artifact以外のArtifactをdynamicにmaterializeした
- 親Adapterが明示 `reuse_eligible=false` としたexecutor extension

理由はruntime中に追加dependencyを読んだため。

通常Recoveryで成功済みJobを単に履歴として保持すること自体は可能だが、**Manual Retryによるdependency変更後にそのsuccessを再利用する場合**はeligibility/key検証必須。

## 12. Reuse判定条件

同一Workflow Run内でのみ、以下がすべて同一なら成功結果をreuse可能:

1. final persistent Job Input
2. direct upstream Artifact identities
3. entire Workflow Definition hash
4. executor identity
5. stored result Payloadが存在しintegrity検証可能
6. current Registryがinternal Actionのsnapshot versionを提供可能
7. `reuse_eligible=true`

Secret materialized valueは保存/比較しない。Secret rotationが結果互換性へ影響する場合は親側Action version/Workflow version管理責任とする。

## 13. Manual Retry後のdescendant処理

`wf_retry(target_failed_job)` でRunをreopenするとき:

### blocked / skipped descendants

Targetのdependency closureに属する `blocked` / `skipped` Jobはterminal stateを解除してactivation再評価対象へ戻す。Explicit conditionも再評価する。

### successful descendants

削除/再実行しない。`reuse_check_pending=true`を付ける。

Dependenciesが再びterminalになった時点で現在contextからreuse keyを再計算:

- match -> successを保持、`job_result_reused` Event、pending clear
- mismatch / ineligible / payload missing -> **その成功結果を自動使用しない**

MVPでは成功済みJobを新Inputで同じJob Run内に自動再実行しない。これは「Retry Input固定」と衝突するため。

代わりにWorkflow Runをfailureへ確定し:

```text
code = successful_job_result_not_reusable
retryable = false
details = affected job(s), changed component summary
```

新しいWorkflow Run開始を要求する。

### failed descendants

Target以外の既存failed Jobは勝手にRetryしない。必要なら個別manual retry。RetryはそのJob元Inputを固定使用する。

## 14. Pause / Resume

Pause:

- Run paused
- running internal継続
- new internal/External claim/Dynamic expansion禁止
- existing External submit/Human review/started Child進行は許可

Resumeでactivation再評価。new Attemptは作らない。

## 15. Cancel

- cancel_requested
- queued/waitingをcancel
- External Lease invalidation
- running internal cancel request
- Child cancel propagation
- new activation禁止

`always()`でもcancel後new Job開始無し。

## 16. Workflow conclusion

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
- `successful_job_result_not_reusable`
- engine fatal

Cancel request由来はcancelled。

## 17. Concurrency / Recovery

Concurrency holder/releaseは`08`。

Runtime restart:

- queued activation再評価
- External Lease/Review/Child関係復元
- running internal -> `10` Runner recovery
- paused維持
- completed success Jobは保持
- pending reuse checksがあれば再開

Recoveryだけでcompleted Runをreopenしない。

## 18. 受入条件

1. internal one-running
2. deterministic ordering
3. pause/cancel/concurrency
4. same-run reuse only
5. reuse key Input/Artifact/definition/action version
6. large/spilled Payload存在check
7. state.get使用Job ineligible
8. dynamic artifact access ineligible
9. Manual Retry blocked/skipped再評価
10. successful descendant match reuse
11. successful descendant mismatch -> new Run required
12. no cross-Run reuse
