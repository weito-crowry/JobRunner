# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v1.2
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `04`, `08`, `11`, `12`

## 1. 目的

ArtifactStore、ArtifactRef、全executor共通Execution Log、Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

## 2. Artifactの2種類

### Managed Artifact

ActionがJobRunnerへ明示保存を依頼した成果物。Core `ArtifactStore`が実体管理。

### External Reference Artifact

親システム等が既に保存した成果物へのURI参照。Coreはmetadataのみ管理しfetch/delete無し。

Attempt temp fileを自動Artifact化しない。残す場合は`put_file`またはexternal reference登録。

## 3. ArtifactStore interface

```text
put_file(artifact_id, source_path) -> store_key,size,digest
materialize(store_key, destination_path)
delete(store_key)
exists(store_key)
```

MVP=`LocalArtifactStore`。Data root `artifacts/`へdurable保存。Backend交換可能。

Managed Artifact digestはCore計算SHA-256で、public ArtifactRefでは `sha256:<lowercase hex64>` とする。

External Reference Artifactの`digest`はoptional。指定する場合MVPでは同じ `sha256:<lowercase hex64>` 形式だけを許可し、Coreは外部実体をfetchしないため値の正しさは検証しない。

## 4. Runtime Handle Artifact

```text
runtime.artifact.put_file(name, relative_work_path, media_type?, metadata?)
runtime.artifact.register_external(name, uri, media_type?, size?, digest?, metadata?)
runtime.artifact.materialize(artifact_ref) -> local_path
```

`put_file`: work_dir内path限定、Secret scan、Store copy、size/digest、metadata/Event。

`register_external`: Core fetch無し。

`materialize`: Managedのみ、§6。

External LLM `task_submit.artifacts` はExternal Referenceのみ。Binary upload無し。

## 5. Canonical ArtifactRef

```json
{
  "type": "jobrunner_artifact",
  "artifact_id": "art_...",
  "name": "dataset",
  "storage_kind": "managed",
  "media_type": "application/json",
  "size_bytes": 1234,
  "digest": "sha256:..."
}
```

Externalのみ`uri`追加可能。

Required=`type=jobrunner_artifact`, artifact_id/name non-empty, storage_kind=`managed|external`。

Caller supplied metadataをSource of Truthにせずartifact_idでDB再resolve。Managed store path非公開。

## 6. Artifact scope

Coreはcross-run Artifactを暗黙探索しない。

Same Run: current Run owned Managed Artifactはdata/metadata/current Actor scope確認後利用可能。

Cross Runは全て必須:

1. canonical ArtifactRefがcurrent persistent Job Inputに存在
2. row/data存在、not deleted
3. source Artifact read Authorization
4. refとDB metadata整合

Raw artifact_idだけでは不可。

ArtifactRefを別Runへ渡してもsource Retention hold/pin無し。Sourceが削除されたらfail-closed。

External ReferenceはCore materialize無し。

## 7. Artifact generations / reuse

Artifact immutable。Same Attempt/name複数generation可。Current successful Attempt内で最新non-deleted generationをcurrentとする。

`needs.<job>.artifacts.<name>` はcanonical ArtifactRef。

Persistent Input内ArtifactRefはinput digestへ固定。Persistent Input外materializeはAttempt reuse ineligible。

## 8. Output Payloadとの違い

Job/Workflow JSON OutputはPayloadStore auto inline/spill。ArtifactはActionが明示作成する別概念。ArtifactRef JSONはInput/Outputへ含められるがArtifact実体は埋め込まない。

## 9. Run directory

```text
jobrunner-data/
├─ artifacts/
└─ runs/<workflow_run_id>/
   ├─ payloads/
   ├─ logs/
   │  └─ <job_run_id>/<attempt_no>.log
   └─ tmp/
      └─ <job_run_id>/<attempt_no>/
```

Filesystem pathはopaque Core IDのみ。

## 10. Execution Log 共通契約

**internal / external_llm / human / reusable の全Attemptが同じExecution Log subsystemを使う。**

- Internal: Runner claim時作成、Action log/stdout/stderr/Runner diagnostic
- External: activation時作成、Task/Lease/submit/validation lifecycle
- Human: activation時作成、Review lifecycle
- Reusable: activation時作成、Child start/status/conclusion lifecycle

Attempt terminalでclose。Recoveryでopen Logを閉じる場合もclose metadataを確定。

全量memory保持禁止。Periodic flush。

どのexecutorでもpersistent Input/Output全bodyをExecution LogへCoreが自動複製しない。Input/Output APIが本文のSource of Truth。Parent/Actionが明示logする場合は通常Log/SecretGuard規則。

### 10.1 Log verbosity

Effective `execution_log_level` は**当該Workflow Runの `effective_settings.execution_log_level` snapshot**を使う。Current System/Workflow source settingsをLog書込時に再参照しない。

`normal`:

- lifecycle
- Action explicit log
- stdout/stderr
- warning/error
- External/Human/Reusable主要state transition

`debug`:

- normal + scheduler/IPC/lease/storage/validation diagnostic metadata
- ID/size/digest/timing等

DebugでもSecret値やInput/Output全bodyをCoreが自動dumpしない。

### 10.2 Read

`wf_log_read`でattempt IDからoffset/tail。External path指定不可。

Retentionでfile削除済みなら空文字を正常Logとして返さず`log_data_unavailable`を返す。

## 11. Event Log

Append-only structured audit。代表:

```text
workflow_started/completed
job_ready/job_started/job_completed
attempt_started/completed
artifact_registered/artifact_data_deleted
state_changed
runner_lost/restarted
external_task_claimed/submitted
human_review_submitted
child_workflow_started/completed
retry_scheduled/manual_retry_requested
job_result_reused
dynamic_expansion_reused
dynamic_index_key_fallback
```

Progress update/全log lineはEventへ複製しない。

Retention audit=`retention_deleted|retention_orphan_cleaned`、owner FK無し、通常event retention外、MVP無期限。

Public Event read=`11 wf_event_list`。

## 12. Step

Step=Job内部の観測単位。

無し=`needs`, Runner allocation, independent Retry, timeout, Artifact ownership。

Open Step crash -> `incomplete`。

## 13. Job Progress

Resolved `progress_mode`はJob Run作成時にsnapshotする。

Resolution:

```text
Job progress.mode specified
  > Workflow Run effective_settings.job_progress_mode
```

Run effective setting自体は`01`のWorkflow setting > Run System baseline > defaultで既に確定済み。Current System/Workflow source settingsを後から再参照しない。

Mode=`auto|explicit|none`。

Action `progress` reportは`04`形式。DBにはlast explicit reportを保存する。

### 13.1 `none`

Public Job progress=`null`。受信progress messageはvalidation後discardしてよい。

### 13.2 `explicit`

Non-terminal:

- reportあり -> last report公開
- report無し -> indeterminate (`current/total=null`)

Terminalになったらlifecycle completionとして`fraction=1.0`を公開。Conclusion成功を意味しない。

### 13.3 `auto`

優先:

1. Reusable Job + Child Workflow progress available -> Child fraction
2. explicit Action progressにtotalあり -> current/total
3. explicit reportがtotal無し -> fraction indeterminate
4. non-terminal -> 0.0
5. terminal -> 1.0

External/Humanはprogress更新APIをMVPに持たないため通常0->1。InternalはAction report利用可能。

ProgressはConclusion判定へ使わない。

## 14. Workflow Progress

Effective `workflow_progress_mode` は当該Workflow Runの `effective_settings.workflow_progress_mode` snapshotを使う。Mode=`auto|none`。

`none` -> public Workflow progress=null。

`auto` -> **現在DBに存在するconcrete Job Run**のeffective Job progress fractionをweight=1で平均する。

- Static non-dynamic Job Runをcount
- Generated Dynamic Job Runをcount
- Reusable Parent Jobは1 JobとしてcountしChild fractionを内部fractionに使える
- Dynamic template virtual group自体はcountしない
- indeterminate non-terminal Jobは0としてaggregate
- terminal Jobはconclusionに関係なく1
- Generated Job追加で分母増加可、percentage一時低下を許容

Known concrete Job=0でRun non-terminalなら0、Run terminalなら1。

Workflow progressはread時に計算可能で、毎updateをEventへ保存しない。

## 15. Workflow state

`env`=literal-only immutable static、Secret禁止。

Mutable:

```text
state.get(name)
state.set(name,value)
```

Persistence:

- `state.set` は呼び出しごとにcurrent value + append-only historyを1 DB transactionで即時commit
- last-write-wins
- Attempt成功時まで保留しない
- Attemptが後でfailure/cancelled/timeout/runner_lostでもrollbackしない
- Historyはrevision + producer Job/Attempt/Step + actor/timeを保持
- `state.get` / `state.set` をRuntime Handleで使ったAttemptは`03`どおりResult Reuse不適格
- CAS/atomic increment/distributed lock無し
- Child Workflowは独立namespace

State persistenceはAuthorization + SecretGuard。

## 16. Temp lifecycle

Attempt execution開始時mkdir、終了後削除。Cleanup failureはJob conclusion不変、warning/Event/maintenance cleanup候補。Tempはsandboxではない。

## 17. Retention

Source=`workflow_runs.retention_policy_json` snapshot。

### 17.1 Execution Log

- basis=Attempt completed/Log close
- running/non-terminal current Log削除無し
- deleted_atを記録

### 17.2 Normal Event

- basis=created_at
- system retention audit除外

### 17.3 Managed Artifact data

Candidate due dateは以下のうちfiniteな最小:

```text
created_at + managed_artifact_data_days
created_at + artifact_metadata_days
owner Run completed_at + run_history_days   # Run subtree削除時の最終上限
```

ただしowner Run non-terminal中はRetention期限だけでdataを削除しない。

意味:

- `managed-artifact-data-days` が先ならdataだけ削除しmetadata rowは残せる
- `artifact-metadata-days` が先ならmetadataを消す前提としてManaged dataも同時/先に削除する
- data delete成功後`data_deleted_at`記録
- cross-run ArtifactRefはpinしない

### 17.4 Artifact metadata

Artifact metadata row due:

```text
created_at + artifact_metadata_days
```

またはowner Run history deletion時。

- owner Run non-terminal中はRetention期限だけでmetadata削除しない
- Managed dataが残っているrowを先に削除しない。必要ならdataを先にdelete
- metadata retention実施時はArtifact row自体を削除する
- row削除後`wf_artifact_info`は`not_found`
- cross-run ArtifactRefはsource row不在でfail-closed
- External Reference Artifact metadata rowを削除しても外部実体は触らない

### 17.5 Run history

- basis=Run completed_at
- non-terminal delete禁止
- component retentionの最終上限
- Output PayloadはRun/Attempt historyと共にdelete
- Parent expiryはChild subtree上限

### 17.6 Orphan cleanup

Owner metadata無しtemp/payload/artifact objectはconsistency cleanup可能。System audit Eventを残す。

Orphan scannerは現在進行中のprepare/finalizeと競合しないよう、Core system config `orphan_cleanup_grace_seconds`（default300、finite>=0）より新しいunowned filesystem objectを削除しない。これはhousekeeping設定でありWorkflow Run semanticsではないためRun snapshot対象外。

## 18. RetentionとReuse

Reuse対象Payload/Managed Artifactが削除済みならsilent reuse禁止。Cross-run ArtifactRefもsource retentionで失効し得る。`03/10` reuse validationでfail-closed。

## 19. Security

- put/materialize path=current work_dir内
- Store key/path Core生成
- metadata/URI SecretGuard
- External URI auto-fetch無し
- cross-run explicit ref + Authorization
- Runtime Handle state/artifact authorization=`12`
- Log Secret redaction

## 20. 受入条件

1. Artifact generations/scope/Authorization
2. Managed/external Artifact digest format
3. cross-run no retention pin
4. common Execution Log all four executor types
5. execution log level uses Run snapshot and survives System/source config change
6. no automatic Input/Output body dump
7. Log retention deleted-state read behavior
8. Event Log + dynamic index fallback/public event read
9. Step observation-only
10. Job progress resolution snapshot
11. Job progress none/explicit/auto
12. Reusable Child progress aggregation
13. External/Human auto 0->1
14. Workflow progress mode uses Run snapshot
15. Workflow auto concrete-job aggregation
16. Dynamic denominator growth/decrease allowed
17. Progress no effect on conclusion
18. state.set immediate durable/nonrollback
19. state.get/state.set reuse ineligible
20. Artifact data vs metadata retention precedence
21. metadata row deletion semantics
22. orphan cleanup grace race protection
23. temp/retention/orphan cleanup
