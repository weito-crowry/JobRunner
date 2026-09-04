# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.7
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02`, `04`, `08`, `11`, `12`

## 1. 目的

ArtifactStore、ArtifactRef、Artifact参照境界、Execution/Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

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

`materialize`はManaged Artifactのみ。許可条件は§7。

## 5. External LLM Artifact

`task_submit.artifacts` はExternal Reference Artifactのみ。MVP task protocol binary upload無し。

## 6. Canonical ArtifactRef

Public/Expression/Inputで使うArtifactRefはJSON object:

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

Externalの場合のみ `uri` を追加可能。

Required:

```text
type = jobrunner_artifact
artifact_id non-empty
name non-empty
storage_kind = managed|external
```

Optional metadata fieldはArtifact rowから生成する。Caller supplied objectの`name/storage_kind/uri/size/digest`を権限や実体のSource of Truthにしない。Coreは`artifact_id`でDB metadataを再読込し、refとの整合を検証する。

Managed `store_key`/filesystem pathはArtifactRefへ露出しない。

## 7. Artifact scope / materialize authorization

Coreはcross-run Artifactを暗黙探索しない。

`runtime.artifact.materialize(ref)` は次を満たすManaged Artifactだけ許可する。

### 7.1 same Workflow Run

Artifact owner `workflow_run_id == current workflow_run_id` なら、current Run内部data flowとして利用可能。ただしdata未削除・Artifact metadata整合・current Actor/AccessScopeのRun accessを満たすこと。

### 7.2 different Workflow Run

Artifact ownerが別Runの場合、以下を**全て**要求する。

1. canonical ArtifactRefがcurrent Jobのpersistent Input JSON tree内に存在する
2. Artifact row/dataが存在し`data_deleted_at IS NULL`
3. current ActorContext/AccessScopeに対してAuthorizationProviderがsource Artifact readを許可
4. refのartifact_idとDB metadataが一致

これにより、単に他Runのartifact_idを知っているだけではmaterializeできない。

Parentが過去Run Artifactを明示Workflow Inputとして渡すこと、Reusable parentがChild `with`へArtifactRefを渡すことは許可される。

### 7.3 External Reference

Coreはmaterializeしない。ArtifactRefの`uri`をAction/親側が解釈するかは親責任。

### 7.4 Reuse eligibility

- persistent Inputに含まれるArtifactRefはInput digestへ固定される
- same-run direct upstream Artifactは`03`のdirect_upstream_artifactsへも固定
- persistent Input外のArtifactをRuntime中にmaterializeした場合は`reuse_eligible=false`

## 8. Current Artifact / generations

Artifact immutable。Retry/re-execution same nameでもnew artifact_id。

`needs.<job>.artifacts.<name>`:

1. current successful Attempt
2. そのAttemptのnon-deleted同名Artifact最新

返す値はCanonical ArtifactRef。

Old generationはmetadata retentionまで履歴保持。Dynamic groupはfull logical `job_key` map。

## 9. Output Payloadとの違い

Job/Workflow JSON Outputは`02/08` PayloadStoreがauto inline/spill。

ArtifactはActionが明示作成する別概念。Large JSON Outputを手動Artifact化する必要無し。

ArtifactRef自体はJSON-compatibleなので、Job/Workflow OutputやInputへ明示的に含められる。Artifact実体をOutput JSONへ埋め込む意味ではない。

## 10. Run directory

```text
jobrunner-data/
├─ artifacts/
└─ runs/<workflow_run_id>/
   ├─ payloads/
   ├─ logs/
   └─ tmp/
```

Filesystem pathはopaque internal ID。

## 11. Execution Log

Attemptごとfile。Runnerがstructured log/stdout/stderr/diagnosticをappend。

Periodic flush。全量memory保持禁止。

`wf_log_read`でattempt IDからfull/offset/tail。`wf_run_info`へ本文無し。External path指定不可。

Log close timeはAttempt terminal時。Recoveryでopen logを閉じる場合もclose metadataを確定する。

## 12. Event Log

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

Retention実施証跡は`08`のsystem-level retention audit Event (`retention_deleted`, `retention_orphan_cleaned`)。Run FK無し、通常`event-days`対象外、MVP無期限保持。

## 13. Step

Step=観測単位。`needs`/Runner allocation/independent Retry/timeout/Artifact ownership無し。

Open Step中crashはAttempt terminal/Recovery時にfailure/incompleteへ閉じる。

## 14. Progress

```text
current >=0
total optional
message/unit optional
```

Workflow progressは表示用。Dynamic展開で分母増加可。Conclusion不使用。

## 15. Workflow state

`env`=literal-only immutable static、Secret禁止。

Mutable:

```text
state.get(name)
state.set(name,value)
```

Persistent、last-write-wins、revision/history。CAS/atomic increment/distributed lock無し。

State persistenceはSecretGuard。Child Workflow独立namespace。

## 16. Temp lifecycle

Attempt execution開始時mkdir、終了後削除。

Cleanup failureはJob conclusion不変、warning/Event/maintenance cleanup候補。

Temp=sandboxではない。

## 17. Retention policy

Source of Truth=`01`の5項目 + `08.workflow_runs.retention_policy_json` snapshot。

```text
run-history-days
execution-logs-days
event-days
artifact-metadata-days
managed-artifact-data-days
```

`null=unlimited`。System default全null。Workflow指定項目がSystemをoverride。

### 17.1 Execution Log

- due基準=Attempt completed/Log close time
- running/non-terminal Attemptのcurrent logは削除しない
- due削除後`wf_log_read`は空Logと偽装しない

### 17.2 Normal Event

- due基準=`created_at`
- `event-days`で削除可能
- system-level retention audit Eventは除外

### 17.3 Managed Artifact data

- due基準=Artifact `created_at`
- owner Workflow Run non-terminal中は期限だけで削除しない
- ArtifactStore delete成功後`data_deleted_at`
- current reference/materializeは`data_deleted_at IS NULL`のみusable

### 17.4 Artifact metadata

- due基準=Artifact `created_at`
- owner Run non-terminal中は削除しない
- Managed dataが残る場合metadata先行削除禁止
- External metadata削除でも外部実体は触らない

### 17.5 Run history

- due基準=Workflow Run `completed_at`
- non-terminal Run削除禁止
- Run row削除時は`08`のFK依存順
- component別retentionが長くてもrun-history expiryが最終上限
- Output Payloadは独立retention無し、Run/Attempt historyと共に削除

### 17.6 Orphan cleanup

Crashでowner metadataを持たないtemp/payload/artifact store objectはMaintenance cleanup可能。

通常Retention policyとは別のconsistency cleanup。system-level `retention_orphan_cleaned` audit Eventを残す。

## 18. RetentionとResult Reuse

Successful descendant reuse対象Payload/Managed ArtifactがRetentionで既に消えている場合、silent reuse禁止。

`03/10`のreuse validationでavailability不一致を検出し `successful_job_result_not_reusable` へ。

## 19. Security

- put source/current materialize destinationはwork_dir内
- Managed store key/path Core生成
- Artifact metadata/URI SecretGuard
- External URI auto-fetch無し
- cross-run materializeはpersistent Input明示 + Authorization必須
- Retention audit payloadもSecretGuard

## 20. 受入条件

1. managed put_file/generations
2. traversal reject/materialize safety
3. canonical ArtifactRef validation/DB re-resolve
4. same-run Artifact materialize
5. explicit cross-run ArtifactRef materialize + authorization
6. raw cross-run artifact_id only reject
7. External Artifact core materialize reject
8. Output Payloadとの分離
9. Execution Log read/no Run info body
10. State history/SecretGuard
11. temp cleanup
12. retention inheritance/cutoff
13. non-terminal retention guard
14. managed data_deleted_at/current reference behavior
15. run-history FK cleanup
16. retention audit survives normal event retention
17. orphan cleanup audit
18. retention loss causes reuse fail-closed
