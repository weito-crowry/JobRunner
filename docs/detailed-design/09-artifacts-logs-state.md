# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `04-runner-and-ipc.md`, `08-persistence.md`

## 1. 目的

Artifact参照管理、Execution/Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

## 2. Artifact責務

1. Artifact実体の保存はAction/親システム責任。
2. Coreはmetadata/生成元/history/referenceを管理。
3. Artifactは論理immutable。
4. CoreがURIをfetch/uploadしない。
5. 小さいJob Output JSONはCore保存。大きいデータはArtifactへ逃がす。

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

`name`は1文字以上。Artifact metadataはJSON-compatible。`size_bytes>=0`。

## 4. 登録

Internal ActionはRuntime Handle、Externalは`task_submit.artifacts`から登録する。

```python
runtime.artifact(
    name="report",
    uri="project://reports/123.json",
    media_type="application/json",
    size_bytes=1234,
    digest="sha256:...",
)
```

CoreはURI実体の存在/内容を必須検証しない。実体保存成功後に登録する責任はAction/親側。

## 5. Current Artifact解決

同名ArtifactはRetry/再実行で複数世代を持てる。Alias tableはMVPでは使わない。

`needs.<job>.artifacts.<name>`:

1. Jobのcurrent successful Attemptを解決
2. そのAttempt内の非deleted同名Artifactを`created_at, artifact_id`の最新順で1件

failed/cancelled AttemptのArtifactをcurrentとして公開しない。履歴APIでは全世代を読める。

## 6. Dynamic Artifact

Dynamic groupは`02/05`のfull logical job_key mapを使う。

```text
needs.evaluate.artifacts["evaluate[candidate_a]"]["report"]
```

Nestedもfull path。

## 7. Artifact deletion / retention

Core retention対象はmetadata。Artifact実体削除は親責任。

metadata削除前に`retention_deleted` Eventを残す。必要なら先に`deleted_at`をセットして参照対象外にし、後続maintenanceで物理row削除できる。

## 8. Workflow Run directory

Core data root:

```text
jobrunner-data/
└─ runs/<workflow_run_id>/
   ├─ logs/
   └─ tmp/
```

Actionの永続共有領域ではない。

## 9. Execution Log path

YAML Job ID/full Dynamic keyをfilesystem pathに使わない。内部IDを使う。

```text
runs/<workflow_run_id>/logs/<job_run_id>/<attempt_no>.log
```

Temp:

```text
runs/<workflow_run_id>/tmp/<job_run_id>/<attempt_no>/
```

DBにはdata rootからのrelative pathのみ保存。PathはCoreが生成し外部入力を連結しない。

## 10. Execution Log write

Runnerが以下を同Attempt logへ追記:

- Action structured `log`
- captured stdout
- captured stderr
- Runnerの必要最小限execution diagnostic

記録形式は人間可読textを基本とし、各行にtimestamp/stream-or-level/step optionalを持てる。

長時間Job用にperiodic/size-based flushし、全量memory bufferは禁止。

## 11. Log read

Serviceはattempt IDで:

```text
metadata
full read
byte offset read
tail lines
```

を提供。`wf_run_info`へ本文を埋め込まない。

外部からfilesystem path指定readは不可。

## 12. Event Log

Append-only structured audit。

代表:

```text
workflow_started/completed/paused/resumed/cancel_requested
job_ready/started/completed
attempt_started/completed
step_started/completed
artifact_registered
state_changed
runner_lost/restarted
external_task_created/claimed/submitted
human_review_requested/submitted
reusable_binding_created/child_workflow_started/completed
retry_scheduled/manual_retry_requested
retention_deleted
```

Common fields:

```text
event_id/type/version/created_at
workflow_run_id/job_run_id/attempt_id/runner_id optional
actor_type/actor_id/source optional
payload
```

Progress/全log lineをEvent tableへ複製しない。

## 13. Step

Stepは観測単位。

```text
step_start(name)
step_end(conclusion)
```

持たないもの:

- `needs`
- Runner割当
- independent Retry/timeout
- Artifact ownership

Actionがopen Step中にcrashしたらAttempt終端時/Recovery時にfailure/incompleteとして閉じる。

## 14. Progress

```text
current >=0
total optional
message/unit optional
```

`total`ありなら`total>0`かつ`current<=total`。indeterminateを許可。

Workflow Progress既定はterminal Job数/既知Job数から表示用に算出可能。Dynamic展開で分母が増えるため割合が下がることを許容する。Conclusion判定には使用しない。

Child Workflow progressをParent waiting_child Jobの表示へ集約可能。

## 15. Workflow static values

YAML `env`はRun start snapshot、immutable。Secret valueは含めない。

## 16. Workflow mutable state

Runtime Handle:

```text
state.get(name)
state.set(name, value)
```

値はJSON-compatible。

Core保証:

- persistence/restart resume
- get/set
- last-write-wins
- revision単調増加
- append-only history

保証しない:

- CAS
- atomic increment
- distributed lock
- read-modify-write race防止

1 Workflow Run internal同時1Jobなので通常競合は少ないが、External/管理操作等を含めlast-write-wins規則を維持する。

## 17. State history

`set` 1transactionでcurrent更新 + history追加。

```text
name
old/new
revision
job_run_id/attempt_id/step_id optional
actor optional
created_at
```

Expression `state.*`はread-only。Job Inputへ解決した値はそのInput snapshotへ固定。

## 18. Child Workflow

Childは独立state namespace。親state直接read/write不可。Input/Outputで渡す。

## 19. Temp lifecycle

Attempt execution開始時にmkdir。終了後success/failure/cancel問わず削除。

Temp cleanup failure:

- Job conclusion変更なし
- warning/Event
- maintenance cleanup候補

Tempはsecurity sandboxではない。

## 20. Retention

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

Schedulerを新設せず、Runtime起動/親maintenance hook/明示Serviceからretention処理を呼べる。

親側Artifact実体の削除方法はCoreが推測しない。

## 21. Log/Secret security

Known Secret redaction hookをwrite pipeline入口で適用する。ただし変形/分割されたSecretを完全検出する保証はしない。

Execution Log readはAuthorization対象。Definition/Input/EventへSecret平文を書かない。

## 22. 受入条件

1. Artifact register/current/history
2. failed Attempt Artifact非current
3. Dynamic full-key Artifact lookup
4. internal/external Artifact登録
5. safe internal-ID log/temp path
6. stdout/stderr/log flush/read tail
7. path traversal不可
8. Step crash close
9. progress indeterminate/dynamic denominator
10. state get/set/history/restart
11. Child state isolation
12. temp cleanup/failure warning
13. retention Event
14. Secret redaction hook
