# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.6
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `04`, `08`, `11`, `12`

## 1. 目的

ArtifactStore、Artifact参照、Execution/Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

## 2. Artifactの2種類

### Managed Artifact

ActionがJobRunnerへ明示保存を依頼した成果物。Core `ArtifactStore`が実体管理。

### External Reference Artifact

親システム等が既に保存した成果物へのURI参照。Coreはmetadataのみ管理し実体fetch/delete無し。

Attempt temp directory内fileを自動Artifact化しない。残す場合は`put_file`またはexternal reference登録。

## 3. ArtifactStore interface

```text
put_file(artifact_id, source_path) -> store_key,size,digest
materialize(store_key, destination_path)
delete(store_key)
exists(store_key)
```

MVP standard=`LocalArtifactStore`。data root配下`artifacts/`へdurable保存。Backend交換可能。

## 4. Runtime Handle

```text
runtime.artifact.put_file(name, relative_work_path, media_type?, metadata?)
runtime.artifact.register_external(name, uri, media_type?, size?, digest?, metadata?)
runtime.artifact.materialize(artifact_ref) -> local_path
```

`put_file`:

- source=current Attempt work_dir相対path
- canonical pathがwork_dir外ならreject
- RunnerがSecret scan/ArtifactStore copy/size/digest
- success後metadata/Event commit

`register_external`: Core fetch無し。

`materialize`: Managedのみcurrent Attempt work_dir配下へ。External referenceはCore標準unsupported。

## 5. External LLM Artifact

`task_submit.artifacts` はExternal Reference Artifactのみ。MVP task protocol binary upload無し。

## 6. Artifact reference shape

```text
artifact_id
name
storage_kind=managed|external
media_type optional
size_bytes optional
digest optional
uri optional  # external only
```

Managed `store_key`/filesystem pathはpublicへ露出しない。

## 7. Current Artifact / generations

Artifact immutable。Retry/re-execution same nameでもnew artifact_id。

`needs.<job>.artifacts.<name>`:

1. current successful Attempt
2. そのAttemptのnon-deleted同名Artifact最新

Old generationはmetadata retentionまで履歴保持。Dynamic groupはfull logical `job_key` map。

## 8. Output Payloadとの違い

Job/Workflow JSON Outputは`02/08` PayloadStoreがauto inline/spill。

ArtifactはActionが明示作成する別概念。Large JSON Outputを手動Artifact化する必要無し。

## 9. Run directory

```text
jobrunner-data/
├─ artifacts/
└─ runs/<workflow_run_id>/
   ├─ payloads/
   ├─ logs/
   └─ tmp/
```

Filesystem pathはopaque internal ID。

## 10. Execution Log

Attemptごとfile。Runnerがstructured log/stdout/stderr/diagnosticをappend。

Periodic flush。全量memory保持禁止。

`wf_log_read`でattempt IDからfull/offset/tail。`wf_run_info`へ本文無し。External path指定不可。

Log close timeはAttempt terminal時。Recoveryでopen logを閉じる場合もclose metadataを確定する。

## 11. Event Log

通常Eventはappend-only structured audit。代表:

```text
workflow_started/completed
job_started/completed
attempt_started/completed
artifact_registered/artifact_data_deleted
state_changed
runner_lost/restarted
external_task_claimed/submitted
human_review_submitted
child_workflow_started/completed
retry_scheduled/manual_retry_requested
```

Progress/全log lineはEvent tableへ複製しない。

Retention実施証跡は`08`の**system-level retention audit Event** (`retention_deleted`, `retention_orphan_cleaned`) を使う。これはRun FKを持たず通常`event-days`対象外、MVP無期限保持。

## 12. Step

Step=観測単位。`needs`/Runner allocation/independent Retry/timeout/Artifact ownership無し。

Open Step中crashはAttempt terminal/Recovery時にfailure/incompleteへ閉じる。

## 13. Progress

```text
current >=0
total optional
message/unit optional
```

Workflow progressは表示用。Dynamic展開で分母増加可。Conclusion不使用。

## 14. Workflow state

`env`=literal-only immutable static、Secret禁止。

Mutable:

```text
state.get(name)
state.set(name,value)
```

Persistent、last-write-wins、revision/history。CAS/atomic increment/distributed lock無し。

State persistenceはSecretGuard。Child Workflow独立namespace。

## 15. Temp lifecycle

Attempt execution開始時mkdir、終了後削除。

Cleanup failureはJob conclusion不変、warning/Event/maintenance cleanup候補。

Temp=sandboxではない。

## 16. Retention policy

Source of Truth=`01`の5項目 + `08.workflow_runs.retention_policy_json` snapshot。

```text
run-history-days
execution-logs-days
event-days
artifact-metadata-days
managed-artifact-data-days
```

`null=unlimited`。System default全null。Workflow指定項目がSystemをoverride。

### 16.1 Execution Log

- due基準=Attempt completed/Log close time
- running/non-terminal Attemptのcurrent logは削除しない
- due削除後`wf_log_read`は存在しないLogを明示`not_found`/retained-away相当metadataとして扱い、空Logと偽装しない

### 16.2 Normal Event

- due基準=`created_at`
- `event-days`で削除可能
- system-level retention audit Eventは除外

### 16.3 Managed Artifact data

- due基準=Artifact `created_at`
- owner Workflow Run non-terminal中は期限だけで削除しない
- ArtifactStore delete成功後`data_deleted_at`
- current reference解決では`data_deleted_at IS NULL`のみusable
- data削除後のmaterializeは明示failure

### 16.4 Artifact metadata

- due基準=Artifact `created_at`
- owner Run non-terminal中は削除しない
- Managed dataが残る場合、metadata先行削除禁止。必要ならdata deleteを先に行う
- External metadata削除でも外部実体は触らない

### 16.5 Run history

- due基準=Workflow Run `completed_at`
- non-terminal Run削除禁止
- Run row削除時は`08`のFK依存順
- component別retentionが長くてもrun-history expiryが最終上限
- Output Payloadは独立retention無し、Run/Attempt historyと共に削除

### 16.6 Orphan cleanup

Crashでowner metadataを持たないtemp/payload/artifact store objectはMaintenance cleanup可能。

通常Retention policyとは別のconsistency cleanup。system-level `retention_orphan_cleaned` audit Eventを残す。

## 17. RetentionとResult Reuse

Successful descendant reuse対象のPayload/Managed ArtifactがRetentionで既に消えている場合、silent reuse禁止。

`03/10`のreuse validationでpayload/artifact availability不一致を検出し `successful_job_result_not_reusable` へ。

## 18. Security

- put source/current materialize destinationはwork_dir内
- Managed store key/path Core生成
- Artifact metadata/URI SecretGuard
- External URI auto-fetch無し
- Retention audit payloadもSecretGuard

## 19. 受入条件

1. managed put_file/generations
2. traversal reject/materialize safety
3. external no fetch/delete
4. Output Payloadとの分離
5. Execution Log read/no Run info body
6. State history/SecretGuard
7. temp cleanup
8. retention inheritance snapshotは`01/08`と一致
9. Log/Event/Artifact data/metadata cutoff
10. non-terminal retention guard
11. managed data_deleted_at/current reference behavior
12. run-history FK cleanup
13. retention audit Event survives normal event retention
14. orphan cleanup audit
15. retention loss causes reuse fail-closed
