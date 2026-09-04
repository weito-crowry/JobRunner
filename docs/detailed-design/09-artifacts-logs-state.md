# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.1
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連:
  - `docs/detailed-design/02-expression-and-inputs.md`
  - `docs/detailed-design/04-runner-and-ipc.md`
  - `docs/detailed-design/08-persistence.md`

## 1. 目的

本書は JobRunner における Artifact 参照管理、Execution Log、Event Log、Workflow共有state、Progress、Step、Workflow Run専用directoryを定義する。

## 2. 基本原則

1. Artifact実体の保存は親システム / Action責任。
2. CoreはArtifact metadataと参照関係を管理する。
3. Job Outputの小さいJSONはCoreで保持する。
4. Execution Log本文はfilesystemに保存する。
5. Event Logはappend-onlyで構造化保存する。
6. Workflow mutable stateはcurrent値とappend-only履歴を両方持つ。
7. Stepは観測単位でありScheduler単位ではない。
8. Job/Attemptの一時作業directoryは終了時に削除する。

## 3. Artifact model

最低限以下を持つ。

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

Artifactは論理的にimmutableとする。

## 4. Artifact登録

ActionはRuntime Handle経由でArtifactを登録する。

概念API:

```python
runtime.artifact(
    name="report",
    uri="project://reports/123.json",
    media_type="application/json",
    size_bytes=1234,
    digest="...",
    metadata={...},
)
```

Coreはuri先の実体内容を必須検査しない。

親システムが保存した後に登録する責任を持つ。

## 5. 同名Artifact

Retryや再実行で同じnameを登録してよい。

古いArtifact rowは残す。

`needs.<job>.artifacts.<name>` はそのJobのcurrent successful Attemptに属する最新有効Artifactを解決する。

履歴取得APIでは全世代を参照可能にする。

## 6. Artifact参照

後段Job:

```yaml
with:
  source: ${{ needs.export.artifacts.dataset }}
```

評価結果は実体ではなく参照object。

例:

```json
{
  "artifact_id": "art_123",
  "name": "dataset",
  "uri": "project://datasets/abc.parquet",
  "media_type": "application/parquet",
  "size_bytes": 123456
}
```

実体readはAction / 親システム責任。

## 7. ArtifactとDynamic Job

Dynamic Job群のArtifactはstable key付きmapとして集約可能。

```yaml
with:
  reports: ${{ needs.evaluate.artifacts.report }}
```

概念結果:

```json
{
  "candidate_a": {"artifact_id": "..."},
  "candidate_b": {"artifact_id": "..."}
}
```

## 8. Artifact deletion

Coreが管理するのはmetadataのretention。

Artifact実体削除は原則親側責任。

metadataがretentionで消える場合もEventを残す。

`deleted_at` を使った論理削除を経由してもよい。

## 9. Execution Log

Job / Attemptごとにfileを持つ。

```text
jobrunner-data/
└─ runs/<workflow_run_id>/logs/<job_key>/<attempt_no>.log
```

DBにはmetadataのみ保存する。

## 10. Log write

RunnerがAction Runnerから受け取ったlog event、stdout、stderrを同じAttempt logへ追記する。

最低限記録可能な属性:

```text
timestamp
stream or level
step optional
message
```

file formatは人間が直接読めるtextを優先する。

構造化補助情報はprefixまたはside metadataで保持してよい。

## 11. stdout / stderr

Actionの通常stdout / stderrはExecution Logへ送る。

JSON Lines protocolと同じstdoutを共有してprotocol破壊しないようにする。

Common Action Runnerがcapturingを担当する。

## 12. Log flush

長時間Jobを考慮し、一定量または一定時間ごとにflushする。

Job終了までbuffer全量をmemory保持しない。

Process crash後も可能な限り直前logが残ることを優先する。

## 13. Log read

Service APIは以下を提供可能にする。

```text
attempt log metadata
full read
byte offset / line offset read
tail read
```

大きいlogを`wf_info`へ埋め込まない。

## 14. Step

StepはActionがRuntime Handleから報告する。

概念:

```python
with runtime.step("load-data"):
    ...
```

または:

```text
step_start(name)
step_end(name, conclusion)
```

## 15. Step制約

Stepは以下を持たない。

- `needs`
- Runner割当
- 独立Retry
- 独立Artifact ownership
- 独立timeout

RetryはJob全体。

Artifact ownerはJob / Attempt。

## 16. Step異常終了

ActionがStep開始後に異常終了した場合、そのStepをincomplete/failureとしてAttempt確定時に閉じる。

Stepが開いたまま残らないようRecovery時にも補正する。

## 17. Progress

Actionは任意にProgressを報告できる。

```text
current
total optional
message optional
unit optional
```

`total`が無いindeterminate progressも許可する。

Progress値は単調増加を推奨するが、Coreはdomain上の再計算を禁止しない。

## 18. Workflow Progress

既定のWorkflow ProgressはJob completionベースで自動算出可能にする。

Dynamic Job生成後は分母が増える場合がある。

Progress表示用の値であり、Workflow conclusion判定には使わない。

Reusable Workflow JobではChild Workflow Progressを親Jobへ集約可能にする。

## 19. Event Log

Eventはappend-only。

代表:

```text
workflow_started
workflow_paused
workflow_resumed
workflow_cancel_requested
workflow_completed
job_queued
job_started
job_completed
attempt_started
attempt_completed
step_started
step_completed
artifact_registered
runner_lost
runner_restarted
external_task_claimed
external_task_submitted
human_review_submitted
state_changed
retention_deleted
```

## 20. Event payload

共通field:

```text
event_id
event_type
created_at
workflow_run_id optional
job_run_id optional
attempt_id optional
runner_id optional
actor_type optional
actor_id optional
source optional
payload
```

Event payloadのschemaはevent_typeごとにversion可能とする。

## 21. Audit性

State-changing Service operationは対応Eventを残す。

ただし大量Progress/logはEvent tableへ1件ずつ保存しない。

Event LogとExecution Logの役割を分離する。

## 22. Workflow static values

YAMLの`env`等はWorkflow Run開始時にsnapshotし、immutableとする。

式contextからread-only参照可能。

## 23. Workflow mutable state

Workflow Run内にkey/value stateを持つ。

概念API:

```text
state.get(name)
state.set(name, value)
```

値はJSON-compatible。

## 24. State semantics

Coreの保証:

- persistent
- restart/resume後も維持
- get/set
- last-write-wins
- revision増加
- history保存

保証しないもの:

- compare-and-swap
- atomic increment
- distributed lock
- read-modify-write race防止

## 25. State history

set時にcurrent更新とhistory追加を同一transactionで行う。

履歴:

```text
name
old_value
new_value
revision
job_run_id optional
attempt_id optional
step_id optional
actor optional
created_at
```

## 26. Child Workflow

Reusable WorkflowのChildは独立stateを持つ。

親stateを直接read/writeしない。

Input / Outputで受け渡す。

## 27. Workflow Run directory

Core管理data root:

```text
jobrunner-data/
└─ runs/<workflow_run_id>/
   ├─ logs/
   └─ tmp/
```

Workflow Run directoryはActionの永続共有領域ではない。

## 28. Attempt temp directory

```text
tmp/<job_key>/<attempt_no>/
```

Attempt開始時作成。

Action Runnerへpathを渡す。

Attempt終了時にsuccess/failure/cancelを問わず原則削除。

## 29. temp削除失敗

削除失敗はJob結果を書き換えない。

運用Event / warningを残し、後続cleanup対象にする。

## 30. Security境界

Temp directoryはsecurity sandboxではない。

Actionが他pathへアクセスできない保証はMVPではしない。

任意コード実行を必要とする親は専用Action内でDocker等を利用する。

## 31. Retention

既定は無期限。

設定対象:

- Execution Log
- Event metadata
- Workflow Run records
- Artifact metadata

Retention Job自体を独立Schedulerにしない。親起動時・定期maintenance hook等から呼べるServiceとして実装可能にする。

## 32. 受入条件

1. Artifact登録/参照
2. 同名Artifact世代管理
3. Retry後current Artifact解決
4. Dynamic Job Artifact集約
5. stdout/stderr log保存
6. crash時log残存
7. Step正常終了
8. Step途中crash補正
9. Progress保存/参照
10. state get/set
11. state history revision
12. restart後state維持
13. Child state非共有
14. temp directory作成削除
15. temp削除失敗warning
16. retention event
