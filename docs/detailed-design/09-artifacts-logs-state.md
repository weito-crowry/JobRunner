# 09. Artifact / Log / Workflow State 詳細設計

- Status: Draft v0.5
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `02-expression-and-inputs.md`, `04-runner-and-ipc.md`, `08-persistence.md`, `11-service-api-and-mcp.md`

## 1. 目的

ArtifactStore、Artifact参照、Execution/Event Log、Workflow state、Progress/Step、Run directory、Retentionを定義する。

## 2. Artifactの2種類

### Managed Artifact

ActionがJobRunnerへ明示的に保存を依頼した成果物。Coreの`ArtifactStore`が実体を管理する。

### External Reference Artifact

親システム等が既に保存した成果物へのURI参照。Coreはmetadataのみ管理し実体をfetch/deleteしない。

**Attempt temp directory内のファイルを自動的にArtifact化しない。** 残したいものはActionが明示的に`put_file`するか、親側へ保存してexternal reference登録する。

## 3. ArtifactStore interface

概念:

```text
put_file(artifact_id, source_path) -> store_key,size,digest
materialize(store_key, destination_path)
delete(store_key)
exists(store_key)
```

Standard MVP backendは`LocalArtifactStore`。data root配下のdurable `artifacts/`へ保存する。

Backend interfaceは将来S3等へ交換可能。

## 4. Runtime Handle

Internal Action:

```text
runtime.artifact.put_file(name, relative_work_path, media_type?, metadata?)
runtime.artifact.register_external(name, uri, media_type?, size?, digest?, metadata?)
runtime.artifact.materialize(artifact_ref) -> local_path
```

### `put_file`

- sourceはAttempt work_dirからのrelative path
- canonicalize後work_dir外へ出るpathはreject
- RunnerがArtifactStoreへcopyし、size/digestを確定
- 保存成功後metadata/Eventをcommit

### `register_external`

CoreはURIをfetchしない。

### `materialize`

Managed ArtifactだけをActionのAttempt work_dir配下へmaterializeしlocal pathを返す。Store backendがremoteでも同じAPIを維持する。

External referenceはCore標準でmaterializeせず、Action/親側がURIを解釈する。

## 5. External LLM Artifact

`task_submit.artifacts` はExternal Reference Artifactのみ登録可能。MVP task protocolにbinary uploadを持たせない。

将来Web upload等を追加する場合もArtifactStoreへ明示putする別APIとする。

## 6. Artifact reference shape

```text
artifact_id
name
storage_kind = managed | external
media_type optional
size_bytes optional
digest optional
uri optional  # externalのみ
```

Managed artifactの`store_key`やfilesystem pathはpublic referenceへ露出しない。

## 7. Current Artifact / generations

Artifactはimmutable。Retry/re-executionで同じnameを登録してもnew artifact_id。

`needs.<job>.artifacts.<name>`:

1. current successful Attempt
2. そのAttemptの非deleted同名Artifactから最新

old generationは履歴に残す。

Dynamic groupはfull logical `job_key` map。

## 8. Artifact retention

Managed:

- retention時ArtifactStore実体をdelete
- metadataに`data_deleted_at`
- 必要な履歴metadataは設定に従い保持可能

External:

- Coreは外部実体をdeleteしない
- metadata retentionのみ

削除Eventを残す。

## 9. Output Payloadとの違い

Job/Workflow JSON Outputは`02/08`のPayloadStoreが自動inline/spillする。

ArtifactはActionが明示的に作る別概念。大きなJSON OutputをActionが手動Artifactへ変換する必要はない。

## 10. Run directory

```text
jobrunner-data/
├─ artifacts/                    # managed durable artifact store
└─ runs/<workflow_run_id>/
   ├─ payloads/                  # durable spilled JSON output
   ├─ logs/
   └─ tmp/
```

Filesystem pathにはopaque internal IDを使用。

## 11. Execution Log

Attemptごとにfile。Runnerがstructured log/stdout/stderr/diagnosticを追記。

Periodic flush。全量memory保持禁止。

`wf_log_read`でattempt IDからfull/offset/tail read。`wf_run_info`に本文を埋め込まない。外部path指定read不可。

## 12. Event Log

Append-only structured audit。代表:

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
retention_deleted
```

Progress/全log lineはEvent tableへ複製しない。

## 13. Step

Stepは観測単位。`needs`/Runner割当/independent Retry/timeout/Artifact ownershipなし。

Open Step中crashはAttempt終端/Recovery時にfailure/incompleteへ閉じる。

## 14. Progress

```text
current >=0
total optional
message/unit optional
```

Workflow progressは表示用。Dynamic展開で分母増加を許容。Conclusionには使わない。

## 15. Workflow state

`env`: literal-only immutable static values。Secret禁止。

Mutable:

```text
state.get(name)
state.set(name,value)
```

Persistent、last-write-wins、revision/history。CAS/atomic increment/distributed lockなし。

State persistenceはSecretGuard通過必須。

Child Workflowは独立state namespace。

## 16. Temp lifecycle

Attempt execution開始時mkdir、終了後削除。

Temp cleanup failureはJob conclusion変更なし、warning/Event/maintenance対象。

Tempはsandboxではない。

## 17. Security

- `put_file` source pathはwork_dir内限定
- Managed store key/pathはCore生成
- Artifact metadata/URIはSecretGuard対象
- Artifact external URIをCoreがfetchしない
- Managed Artifact materialize destinationもcurrent work_dir内限定

## 18. 受入条件

1. managed put_file + immutable generations
2. work_dir traversal reject
3. managed materialize
4. external reference no fetch/delete
5. External task artifactはreference only
6. Output PayloadとArtifactの分離
7. managed retention data delete
8. Dynamic full-key reference
9. Log read/path safety / `wf_run_info` no log body
10. state/history/SecretGuard
11. temp cleanup
