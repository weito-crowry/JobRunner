# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.3
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `04-runner-and-ipc.md`, `08-persistence.md`

## 1. 目的

Artifact参照管理、Execution/Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

## 2. Artifact責務

1. Artifact実体の保存はAction/親システム責任。
2. Coreはmetadata/生成元/history/referenceを管理。
3. Artifactは論理immutable。
4. CoreはURIをfetch/uploadしない。
5. 小さいJob Output JSONはCore保存。大きいデータはArtifactへ。

## 3. Artifact model

```text
artifact_id
workflow_run_id
job_run_id
attempt_id
name
uri
media_type optional
size_bytes optional
digest optional
metadata optional
created_at
deleted_at optional
```

## 4. 登録

Internal ActionはRuntime Handle、Externalは`task_submit.artifacts`から登録。

CoreはURI実体を必須検査しない。実体保存成功後に登録する責任はAction/親側。

## 5. Current Artifact

`needs.<job>.artifacts.<name>`:

1. Jobのcurrent successful Attempt
2. そのAttemptの非deleted同名Artifactから `created_at, artifact_id` 最新

failed/cancelled AttemptのArtifactはcurrentにしない。履歴APIでは全世代参照可能。

## 6. Dynamic Artifact

full logical `job_key` mapを使う。

```text
needs.evaluate.artifacts["evaluate[candidate_a]"]["report"]
```

## 7. Artifact retention

Core retention対象はmetadata。Artifact実体削除は親責任。

metadata削除前に `retention_deleted` Event。必要なら `deleted_at` で論理削除後に物理row削除。

## 8. Run directory

```text
jobrunner-data/
└─ runs/<workflow_run_id>/
   ├─ logs/
   └─ tmp/
```

Actionの永続共有領域ではない。

## 9. Log / temp path

YAML Job ID/full Dynamic keyをfilesystem pathへ使わず内部IDを使う。

```text
runs/<workflow_run_id>/logs/<job_run_id>/<attempt_no>.log
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

DBにはdata rootからのrelative path。外部入力pathを連結しない。

## 10. Execution Log

Runnerが:

- structured log
- stdout
- stderr
- 必要最小限execution diagnostic

をAttempt logへ追記。

長時間Job向けにperiodic/size-based flushし、全量memory保持禁止。

## 11. Log read

Service `wf_log_read` はattempt IDで:

```text
metadata
full read
offset read
tail read
```

を提供。

**`wf_info`へExecution Log本文を埋め込まない。** 外部filesystem path指定readは不可。

## 12. Event Log

append-only structured audit。

代表:

```text
workflow_started
workflow_completed
workflow_paused
workflow_resumed
workflow_cancel_requested
job_ready
job_started
job_completed
attempt_started
attempt_completed
step_started
step_completed
artifact_registered
state_changed
runner_lost
runner_restarted
external_task_created
external_task_claimed
external_task_submitted
human_review_requested
human_review_submitted
reusable_binding_created
child_workflow_started
child_workflow_completed
retry_scheduled
manual_retry_requested
retention_deleted
```

Progress/全log lineをEvent tableへ複製しない。

## 13. Step

Stepは観測単位。`needs` / Runner割当 / independent Retry/timeout / Artifact ownershipを持たない。

open Step中にcrashした場合、Attempt終端/Recovery時にfailure/incompleteとして閉じる。

## 14. Progress

```text
current >= 0
total optional
message/unit optional
```

`total`ありなら `total>0 && current<=total`。

Workflow Progressは表示用。Dynamic展開で分母増加により割合が下がることを許容。Conclusionには使わない。

## 15. Workflow static/mutable values

`env` はRun start snapshot、immutable、Secret参照禁止。

Mutable state:

```text
state.get(name)
state.set(name, value)
```

Core保証:

- persistence/restart
- get/set
- last-write-wins
- revision
- append-only history

保証しない: CAS / atomic increment / distributed lock / read-modify-write race防止。

## 16. State history

`set` 1transactionでcurrent update + history insert。

Expression `state.*`はread-only。Job Inputへ解決した値はsnapshot。

Child Workflowは独立state namespace。

## 17. Temp lifecycle

Attempt execution開始時mkdir、終了後success/failure/cancel問わず削除。

削除失敗:

- Job conclusion変更なし
- warning/Event
- maintenance cleanup候補

Tempはsandboxではない。

## 18. Retention

既定無期限。

対象:

```text
Workflow Run records
Event metadata
Execution Logs
Artifact metadata
Idempotency expired records
orphan temp/log metadata
```

独立Schedulerを必須にせずRuntime起動/maintenance hook/明示Serviceから実行。

## 19. Secret / security

Known Secret redaction hookをwrite pipeline入口で適用。ただし変形Secretの完全検出は保証しない。

Execution Log readはAuthorization対象。Definition/Input/EventへSecret平文を書かない。

## 20. 受入条件

1. Artifact current/history
2. failed Attempt Artifact非current
3. Dynamic full-key lookup
4. safe internal-ID path
5. `wf_info`にLog本文無し
6. stdout/stderr/flush/tail
7. Step crash close
8. state get/set/history/restart
9. env Secret拒否
10. temp cleanup
11. retention Event
