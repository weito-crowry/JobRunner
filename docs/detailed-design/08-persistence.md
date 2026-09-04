# 08. Persistence 詳細設計

- Status: Draft v2.8
- 対象: MVP
- 上位仕様: `docs/design.md`
- Canonical JSON: `01` の `jobrunner.canonical-json.v1`

## 1. 目的

SQLite schema、PayloadStore / ArtifactStore metadata、Input snapshot、System baseline、Dynamic expansion、Reusable binding、Result Reuse、deadline、Retention、Migration、Recovery、Idempotencyを正規契約として定義する。

## 2. Storage構成

```text
SQLite
├─ Runtime state/history/inline JSON
├─ Workflow Definition/System/effective settings snapshot
├─ persistent Job Input/Secret binding metadata
├─ Payload metadata
├─ Artifact metadata
├─ Action/Validator identity
└─ Dynamic/Reusable/Reuse/idempotency/retention metadata

PayloadStore
└─ large JSON Output blob

ArtifactStore
└─ explicitly managed artifact data
```

MVP standard=`sqlite3`, `LocalHybridPayloadStore`, `LocalArtifactStore`。

## 3. SQLite共通規則

```text
PRAGMA foreign_keys=ON
PRAGMA busy_timeout=5000
PRAGMA journal_mode=WAL
```

- 必要時`BEGIN IMMEDIATE`
- 長transaction禁止
- FK原則`ON DELETE NO ACTION`
- implicit cascade無し
- Text identity=BINARY semantics
- ID=type prefix + UUID4 hex TEXT
- INTEGER外部入力=signed64
- JSON/digest=canonical-json-v1
- boolean=INTEGER 0|1

DB columnのCHECKで全runtime invariantを重複実装することは要求しない。以下で明示するenum/invariantのうちSQL CHECK/indexとして記載していないものはRepository/Service transactionでfail-closed検証し、`13` persistence invariant testで固定する。

### 3.1 Canonical timestamp

SQLiteへ永続化する時刻TEXTは全てUTCの固定形式:

```text
YYYY-MM-DDTHH:MM:SS.ffffffZ
```

- fractional secondは常に6桁
- offset表記は使わず`Z`のみ
- timezone-naive値禁止
- lexical order = chronological orderとしてindex/range比較に使用可能
- `now >= deadline/expires_at/retry_not_before` をdue/expired境界とする

Public APIも同じ表現をcanonical serializationとする。

RetentionはFKを無効化せず明示順序。

## 4. Canonical table set / Dynamic template persistence

MVP tableは**18個**。

```text
schema_migrations
workflow_runs
job_runs
job_attempts
job_steps
dynamic_expansions
reusable_bindings
workflow_state
workflow_state_history
artifacts
events
execution_logs
runners
runner_restarts
external_tasks
external_leases
human_reviews
idempotency_records
```

Workflow Definition履歴table無し。実行済みDefinition=`workflow_runs` snapshot。

Dynamic template自体の`job_runs` rowは作らない。

- Run start: static non-dynamic Jobだけ`job_runs`
- Dynamic template definition=`workflow_runs.definition_json`
- Expansion: generated concrete Jobだけ`job_runs`
- Dynamic group state=Definition + `dynamic_expansions` + generated Jobs
- TemplateはRunner/Attempt/Retry/Progress上の実行Jobとして数えない

## 5. Payload column set

Workflow/Attempt Output共通:

```text
output_storage_kind TEXT NULL       # inline|blob
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
```

Invariant:

```text
none: all NULL
inline: kind=inline,json non-NULL,blob NULL,size>=0,digest non-empty
blob: kind=blob,json NULL,blob non-empty,size>=0,digest non-empty
```

## 6. `schema_migrations`

```text
version INTEGER PRIMARY KEY CHECK(version>=1)
name TEXT NOT NULL CHECK(length(name)>0)
applied_at TEXT NOT NULL
```

## 7. `workflow_runs`

```text
id TEXT PRIMARY KEY
workflow_id TEXT NOT NULL CHECK(length(workflow_id)>0)
workflow_version INTEGER NOT NULL CHECK(workflow_version>=1)
name TEXT NOT NULL CHECK(length(name)>0)
status TEXT NOT NULL
conclusion TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
run_attempt INTEGER NOT NULL DEFAULT 1 CHECK(run_attempt>=1)
wait_reason TEXT NULL
pause_requested INTEGER NOT NULL DEFAULT 0 CHECK(pause_requested IN(0,1))
cancel_requested INTEGER NOT NULL DEFAULT 0 CHECK(cancel_requested IN(0,1))
definition_yaml TEXT NOT NULL
definition_json TEXT NOT NULL
definition_hash TEXT NOT NULL CHECK(length(definition_hash)>0)
input_json TEXT NOT NULL
action_versions_json TEXT NOT NULL
validator_versions_json TEXT NOT NULL
system_workflow_defaults_json TEXT NOT NULL
effective_settings_json TEXT NOT NULL
retention_policy_json TEXT NOT NULL
output_storage_kind TEXT NULL
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
failure_json TEXT NULL
source_identity TEXT NULL
concurrency_group TEXT COLLATE BINARY NULL
concurrency_max_runs INTEGER NULL
concurrency_on_limit TEXT NULL
concurrency_queued_at TEXT NULL
actor_context_json TEXT NOT NULL
access_scope_json TEXT NOT NULL
parent_workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
parent_job_run_id TEXT NULL
parent_attempt_id TEXT NULL
root_workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
call_depth INTEGER NOT NULL DEFAULT 0 CHECK(call_depth>=0)
reusable_binding_id TEXT NULL
created_at TEXT NOT NULL
started_at TEXT NULL
completed_at TEXT NULL
updated_at TEXT NOT NULL
```

Logical pointers=`parent_job_run_id/parent_attempt_id/reusable_binding_id`。Repositoryがsame relationをtransaction内検証。

Enums/invariants:

```text
status=queued|running|paused|completed
conclusion=NULL|success|failure|cancelled
wait_reason=NULL|concurrency
queued    -> conclusion NULL, wait_reason=concurrency, concurrency_queued_at non-NULL, slot無し
running   -> conclusion NULL, wait_reason=NULL, concurrency_queued_at NULL, started_at non-NULL,
             concurrency設定時はslot holder
paused    -> conclusion NULL, wait_reason=NULL|concurrency
             wait_reason=NULLならadmitted holder、concurrency_queued_at NULL、started_at non-NULL
             wait_reason=concurrencyならwaiter、concurrency_queued_at non-NULL、slot無し
completed -> conclusion non-NULL, completed_at non-NULL, wait_reason=NULL, concurrency_queued_at NULL
non-completed -> conclusion NULL AND completed_at NULL
source_identity NULL or non-empty
concurrency_group NULL <=> max/on_limit NULL
concurrency_group NULL -> concurrency_queued_at NULL
concurrency_max_runs NULL or >=1
concurrency_on_limit NULL|queue|reject
root: parent_workflow_run_id NULL,depth=0,root_workflow_run_id=id
child: parent ids/binding non-empty,depth>=1
```

### 7.1 Workflow Run timestamp semantics

```text
created_at   = Workflow Run row作成時刻。作成後は不変
started_at   = そのRunが初めてConcurrency admissionされstatus=runningになった時刻
completed_at = current terminal completion時刻。non-terminal中はNULL
```

Rules:

- initial start transactionでadmission成功/no concurrency -> `started_at=created_at`
- initial/Child startでConcurrency queue -> `started_at=NULL`
- waiterが後日slot取得 -> `started_at IS NULL`ならadmission transaction時刻を設定
- `started_at`は一度設定したらPause/Resume/Manual Retry reopen/Recoveryで書き換えない
- admitted `running` / admitted `paused` はstarted_at non-NULL
- 一度もadmitされないwaiterをCancelしてcompleted/cancelledにした場合は`started_at=NULL`を許可
- Manual Retry reopenは`completed_at=NULL`へclearするが`started_at`を保持する
- terminal化時は新しい`completed_at`を設定する
- `workflow_started` Eventは`started_at`初回設定transactionでexactly once記録する

MVPではWorkflow Run `queued` はConcurrency admission待ち専用。Job dependency待ちやRunner待ちだけを理由にWorkflow Runをqueuedへしない。

`concurrency_queued_at`は**実際にConcurrency待ちへ入った時刻**。Run `created_at`の代用にしない。

- initial/Child queue -> queue admission失敗時のcurrent canonical UTC time
- Manual Retry completed Run reopen + queue -> retry request transaction time
- paused waiter ->値を保持
- waiter Resume ->値を保持
- slot admission/completion/cancel terminalization -> NULL
- Recovery ->保存済み値を保持し、current timeへ書き換えない

`system_workflow_defaults_json` exact shape=`01 §11.1`。

Root Runはstart時current System workflow defaultsをsnapshot。Child RunはReusable bindingの`child_system_workflow_defaults_json`をexact copyする。Runtime restart/Retryでcurrent System configから再計算しない。

`effective_settings_json` exact keys:

```text
default_runner_pool
max_dynamic_jobs
external_lease_minutes
external_on_lease_expiry
output_inline_threshold_bytes
execution_log_level
workflow_progress_mode
job_progress_mode
```

Retention policy exact keys=`01`の5項目。

Priority invariant:

- Root initial priority=`wf_start.priority` override or Definition priority
- Child priority=current root Run priority
- Root priority updateはroot + all non-terminal descendantsを同値へ更新
- Child direct priority update無し

Indexes:

```sql
CREATE INDEX ix_workflow_runs_list ON workflow_runs(created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_by_workflow ON workflow_runs(workflow_id,created_at DESC,id ASC);
CREATE INDEX ix_workflow_runs_status ON workflow_runs(status,priority DESC,created_at ASC,id ASC);
CREATE INDEX ix_workflow_runs_concurrency
ON workflow_runs(workflow_id,concurrency_group,status)
WHERE concurrency_group IS NOT NULL AND status!='completed';
CREATE INDEX ix_workflow_runs_concurrency_wait
ON workflow_runs(workflow_id,concurrency_group,priority DESC,concurrency_queued_at ASC,id ASC)
WHERE wait_reason='concurrency' AND status='queued';
CREATE INDEX ix_workflow_runs_root_active ON workflow_runs(root_workflow_run_id,status,call_depth,id)
  WHERE status!='completed';
CREATE UNIQUE INDEX uq_child_run_per_parent_attempt ON workflow_runs(parent_attempt_id)
  WHERE parent_attempt_id IS NOT NULL;
```

## 8. `job_runs`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_key TEXT NOT NULL CHECK(length(job_key)>0)
job_template_key TEXT NOT NULL CHECK(length(job_template_key)>0)
dynamic_key TEXT NULL
parent_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION DEFERRABLE INITIALLY DEFERRED
dynamic_expansion_id TEXT NULL
item_snapshot_json TEXT NULL
iteration_context_json TEXT NULL
source_order INTEGER NOT NULL DEFAULT 0 CHECK(source_order>=0)
order_rank INTEGER NULL CHECK(order_rank IS NULL OR order_rank>=0)
executor TEXT NOT NULL
action_id TEXT NULL
action_version TEXT NULL
validator_id TEXT NULL
validator_version TEXT NULL
uses_ref TEXT NULL
reusable_binding_id TEXT NULL
runs_on TEXT NULL
status TEXT NOT NULL
conclusion TEXT NULL
terminal_reason TEXT NULL
priority INTEGER NOT NULL DEFAULT 0
continue_on_error INTEGER NOT NULL DEFAULT 0 CHECK(continue_on_error IN(0,1))
timeout_seconds REAL NULL
retry_policy_json TEXT NOT NULL
retry_not_before TEXT NULL
ready_at TEXT NULL
pending_input_json TEXT NULL
pending_secret_bindings_json TEXT NULL
pending_input_digest TEXT NULL
current_attempt_no INTEGER NOT NULL DEFAULT 0 CHECK(current_attempt_no>=0)
current_attempt_id TEXT NULL
reuse_check_pending INTEGER NOT NULL DEFAULT 0 CHECK(reuse_check_pending IN(0,1))
current_failure_json TEXT NULL
progress_mode TEXT NOT NULL
progress_current REAL NULL
progress_total REAL NULL
progress_message TEXT NULL
progress_unit TEXT NULL
progress_updated_at TEXT NULL
queued_at TEXT NULL
started_at TEXT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(workflow_run_id,job_key)
```

Logical pointers=`dynamic_expansion_id/reusable_binding_id/current_attempt_id`。

Enums/invariants:

```text
executor=internal|external_llm|human|reusable
status=queued|running|waiting_external|waiting_review|waiting_child|completed
conclusion=NULL|success|failure|cancelled|skipped|blocked
progress_mode=auto|explicit|none
completed <=> conclusion non-NULL
status!=completed -> conclusion NULL AND completed_at NULL
```

Executor invariants:

```text
internal: action/version/runs_on non-empty,uses_ref NULL
external: action/version/runs_on/uses_ref NULL
human: action/version/validator/runs_on/uses_ref NULL
reusable: action/version/validator/runs_on NULL,uses_ref non-empty
validator pair both NULL or both non-empty
timeout only internal,finite>0
```

Internal `runs_on`はJob row作成時にexplicit値またはRun `effective_settings.default_runner_pool`へ解決済みsnapshotを保存する。

Static Jobの`source_order`はMVPでは0。YAML declaration order自体をScheduling保証にせず、`03`のstable `job_key` tie-breakを使う。Generated Jobはforeach source indexを`source_order`へ保存する。

### 8.1 Readiness / pending Input

`pending_*`=次Attempt用snapshot。

```text
status=queued AND ready_at IS NULL -> pending_* all NULL
status=queued AND ready_at IS NOT NULL -> pending_* all non-NULL
status IN(running,waiting_external,waiting_review,waiting_child,completed) -> pending_* all NULL
```

- static initial dependency wait: queued/ready NULL/pending無し
- internal initial activation: queued/ready/pending有り
- external/human/reusable初回:直接Attempt
- all executor Retry: queued/ready/pending有り
- Retry pending作成時は`conclusion=NULL, completed_at=NULL`
- executor-specific activation consumes pending -> Attempt exact copy -> clear

`pending_secret_bindings_json`=`02` canonical array。Secret value無し。

`current_attempt_id`はlatest created Attempt。Retry queued中はprior failed Attemptを指してよい。`current_failure_json`はobservability用にprior failed Attempt failureを保持してよいが、queued Jobの`conclusion`とは別物。

Indexes:

```sql
CREATE UNIQUE INDEX uq_one_running_internal_per_run
ON job_runs(workflow_run_id)
WHERE status='running' AND executor='internal';

CREATE INDEX ix_job_ready_internal
ON job_runs(runs_on,status,retry_not_before,priority DESC,ready_at ASC,job_key ASC,id ASC)
WHERE executor='internal' AND status='queued' AND ready_at IS NOT NULL;

CREATE INDEX ix_job_ready_noninternal
ON job_runs(executor,status,retry_not_before,ready_at,job_key,id)
WHERE executor!='internal' AND status='queued' AND ready_at IS NOT NULL;

CREATE INDEX ix_job_by_workflow ON job_runs(workflow_run_id,source_order ASC,job_key ASC,id ASC);
CREATE INDEX ix_job_retry_due ON job_runs(retry_not_before)
WHERE retry_not_before IS NOT NULL AND status='queued';
CREATE INDEX ix_job_reuse_pending ON job_runs(workflow_run_id,reuse_check_pending)
WHERE reuse_check_pending=1;
```

## 9. `job_attempts`

```text
id TEXT PRIMARY KEY
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_no INTEGER NOT NULL CHECK(attempt_no>=1)
status TEXT NOT NULL
conclusion TEXT NULL
runner_id TEXT NULL
runner_instance_id TEXT NULL
runtime_instance_id TEXT NULL
input_json TEXT NOT NULL
secret_bindings_json TEXT NOT NULL
input_digest TEXT NOT NULL CHECK(length(input_digest)>0)
output_storage_kind TEXT NULL
output_json TEXT NULL
output_blob_key TEXT NULL
output_size_bytes INTEGER NULL
output_digest TEXT NULL
reuse_eligible INTEGER NULL CHECK(reuse_eligible IN(0,1))
reuse_context_json TEXT NULL
reuse_key TEXT NULL
failure_json TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
created_at TEXT NOT NULL
UNIQUE(job_run_id,attempt_no)
```

Status=`running|waiting_external|waiting_review|waiting_child|completed`、Conclusion=`NULL|success|failure|cancelled`。

Pre-Attempt failureはrow無し。

Internal claim=Job pending exact copy。External/Human/Reusable初回=直接snapshot。Retry=全executor pending経由。Secret禁止executor bindings=`[]`。

Successful Attemptで`secret_bindings_json != []`なら`reuse_eligible=0`。Secret valueは保存しないため同一性を証明せず、古いOutputを自動再利用しない。

```sql
CREATE INDEX ix_attempts_job ON job_attempts(job_run_id,attempt_no DESC);
```

## 10. `job_steps`

```text
id TEXT PRIMARY KEY
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
sequence INTEGER NOT NULL CHECK(sequence>=1)
name TEXT NOT NULL CHECK(length(name)>0)
status TEXT NOT NULL
conclusion TEXT NULL
start_metadata_json TEXT NULL
finish_metadata_json TEXT NULL
started_at TEXT NOT NULL
completed_at TEXT NULL
UNIQUE(attempt_id,sequence)
```

Status=`running|completed`, conclusion=`NULL|success|failure|cancelled|incomplete`。

- `step_started`で`start_metadata_json`を保存
- `step_finished`で`finish_metadata_json`を保存
- start metadataをfinishで上書きしない
- telemetry metadataはSecret redaction後JSON-compatible objectだけ保存
- MVPは1 Attemptにつき同時open Step最大1

```sql
CREATE UNIQUE INDEX uq_one_running_step_per_attempt
ON job_steps(attempt_id)
WHERE status='running';
```

## 11. `dynamic_expansions`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
template_id TEXT NOT NULL CHECK(length(template_id)>0)
parent_generated_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
outcome TEXT NOT NULL
source_snapshot_json TEXT NULL
source_digest TEXT NULL
expansion_digest TEXT NULL
generated_count INTEGER NOT NULL DEFAULT 0 CHECK(generated_count>=0)
failure_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
```

`outcome=pending|expanded|skipped|failed|cancelled`。

Invariants:

- pending -> completed_at NULL
- terminal -> completed_at non-NULL
- expanded -> source_snapshot/source_digest/expansion_digest non-NULL
- skipped|cancelled -> generated_count=0
- failed -> failure_json non-NULL

```sql
CREATE UNIQUE INDEX uq_dynamic_expansion_root
ON dynamic_expansions(workflow_run_id,template_id)
WHERE parent_generated_job_run_id IS NULL;
CREATE UNIQUE INDEX uq_dynamic_expansion_nested
ON dynamic_expansions(workflow_run_id,template_id,parent_generated_job_run_id)
WHERE parent_generated_job_run_id IS NOT NULL;
```

Generated Jobs + source/digest/outcome=1 transaction。Expanded rowはsame Runでgenerated set不変。

## 12. `reusable_bindings`

```text
id TEXT PRIMARY KEY
parent_workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
parent_job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
workflow_ref_original TEXT NOT NULL CHECK(length(workflow_ref_original)>0)
child_workflow_id TEXT NOT NULL CHECK(length(child_workflow_id)>0)
child_workflow_version INTEGER NOT NULL CHECK(child_workflow_version>=1)
child_definition_yaml TEXT NOT NULL
child_definition_json TEXT NOT NULL
child_definition_hash TEXT NOT NULL CHECK(length(child_definition_hash)>0)
child_action_versions_json TEXT NOT NULL
child_validator_versions_json TEXT NOT NULL
child_system_workflow_defaults_json TEXT NOT NULL
child_effective_settings_json TEXT NOT NULL
child_retention_policy_json TEXT NOT NULL
created_at TEXT NOT NULL
UNIQUE(parent_job_run_id)
```

`child_system_workflow_defaults_json`=Parent Run `system_workflow_defaults_json` exact copy。

Child effective settings/Retentionはそのbaseline + Child Definition settingsから算出しbinding時固定。Parent Retry/new Childでは再計算しない。

Child Run作成時:

- `system_workflow_defaults_json` <- binding child_system_workflow_defaults_json
- `effective_settings_json` <- binding child_effective_settings_json
- `retention_policy_json` <- binding child_retention_policy_json
- priority <- current root Run priority
- row creation -> `created_at=current time`
- slot admitted -> `status=running, concurrency_queued_at=NULL, started_at=created_at`
- concurrency queue -> `status=queued, wait_reason=concurrency, concurrency_queued_at=created_at, started_at=NULL`

Concurrency `on-limit=reject`でChildを作らない場合もbinding/Parent Attempt failureは保存してよいが、`workflow_runs` Child rowは作らない。

## 13. `workflow_state` / `workflow_state_history`

Current:

```text
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
revision INTEGER NOT NULL CHECK(revision>=1)
value_json TEXT NOT NULL
updated_at TEXT NOT NULL
PRIMARY KEY(workflow_run_id,name)
```

History:

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
revision INTEGER NOT NULL CHECK(revision>=1)
old_value_json TEXT NULL
new_value_json TEXT NOT NULL
job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
step_id TEXT NULL REFERENCES job_steps(id) ON DELETE NO ACTION
actor_json TEXT NULL
created_at TEXT NOT NULL
UNIQUE(workflow_run_id,name,revision)
```

`state.set`=history insert + current upsert same transaction。Last-write-wins。Attempt terminal結果とは独立して即時commitし、後続failure/cancelでrollbackしない。

Runtime Handle `state.set`時点でopen Stepがある場合はそのStep IDを`step_id`へ保存し、open Step無しならNULL。MVPは1 Attempt同時open Step最大1なので対応は一意。

## 14. `artifacts`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
name TEXT NOT NULL CHECK(length(name)>0)
storage_kind TEXT NOT NULL
store_key TEXT NULL
external_uri TEXT NULL
media_type TEXT NULL
size_bytes INTEGER NULL CHECK(size_bytes IS NULL OR size_bytes>=0)
digest TEXT NULL
metadata_json TEXT NULL
created_at TEXT NOT NULL
data_deleted_at TEXT NULL
```

`storage_kind=managed|external`。Managed=store_key only、External=URI only。Same Attempt/name generation可。Current=`created_at DESC,id DESC`。Cross-run pin無し。

Artifact metadata retentionでrow自体を削除するため`metadata_deleted_at` soft-delete columnはMVPに持たない。削除監査はEventで行う。

## 15. `events`

```text
id TEXT PRIMARY KEY
type TEXT NOT NULL CHECK(length(type)>0)
version INTEGER NOT NULL DEFAULT 1 CHECK(version>=1)
workflow_run_id TEXT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
runner_id TEXT NULL
actor_type TEXT NULL
actor_id TEXT NULL
source TEXT NULL
request_id TEXT NULL
payload_json TEXT NOT NULL
created_at TEXT NOT NULL
```

Retention audit=`retention_deleted|retention_orphan_cleaned` owner IDs NULL、通常event retention外・無期限。

```sql
CREATE INDEX ix_events_workflow ON events(workflow_run_id,created_at DESC,id DESC);
CREATE INDEX ix_events_job ON events(job_run_id,created_at DESC,id DESC);
CREATE INDEX ix_events_attempt ON events(attempt_id,created_at DESC,id DESC);
CREATE INDEX ix_events_type_time ON events(type,created_at DESC,id DESC);
```

## 16. `execution_logs`

全executor Attemptに最大1 logical Log。

```text
id TEXT PRIMARY KEY
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
relative_path TEXT NOT NULL CHECK(length(relative_path)>0)
size_bytes INTEGER NOT NULL DEFAULT 0 CHECK(size_bytes>=0)
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
closed_at TEXT NULL
deleted_at TEXT NULL
UNIQUE(attempt_id)
```

Internal=claim時、External/Human/Reusable=Attempt作成時。

## 17. `runners`

```text
runner_instance_id TEXT PRIMARY KEY
runtime_instance_id TEXT NOT NULL
runner_id TEXT NOT NULL
pool_name TEXT NOT NULL CHECK(length(pool_name)>0)
pid INTEGER NOT NULL CHECK(pid>0)
status TEXT NOT NULL
current_job_run_id TEXT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
current_attempt_id TEXT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
started_at TEXT NOT NULL
last_heartbeat_at TEXT NULL
main_loop_tick_at TEXT NULL
stopped_at TEXT NULL
updated_at TEXT NOT NULL
```

Status=`starting|idle|claiming|running|stopping|stopped|lost|restart_suppressed`。

- running -> current Job/Attempt non-NULL
- idle|stopped|lost|restart_suppressed -> current pointers NULL

```sql
CREATE UNIQUE INDEX uq_runner_current_slot
ON runners(runtime_instance_id,pool_name,runner_id)
WHERE status IN('starting','idle','claiming','running','stopping');
CREATE INDEX ix_runners_liveness
ON runners(runtime_instance_id,status,last_heartbeat_at);
```

## 18. `runner_restarts`

```text
id TEXT PRIMARY KEY
runtime_instance_id TEXT NOT NULL
pool_name TEXT NOT NULL
runner_id TEXT NOT NULL
exited_runner_instance_id TEXT NULL
exit_code INTEGER NULL
reason TEXT NOT NULL
restart_ordinal INTEGER NOT NULL CHECK(restart_ordinal>=1)
scheduled_for TEXT NULL
started_runner_instance_id TEXT NULL
suppressed INTEGER NOT NULL DEFAULT 0 CHECK(suppressed IN(0,1))
created_at TEXT NOT NULL
```

Semantics=`04 §22`。

- 1 Runner failureにつきdecision row exactly 1
- `restart_ordinal`は許可/抑止に関係なく、そのfailure時点の`next_restart_ordinal`を保存
- `suppressed=1` -> `scheduled_for=NULL`, `started_runner_instance_id=NULL`
- allowed decision (`suppressed=0`) -> `scheduled_for` non-NULL。Spawn開始成功後`started_runner_instance_id`設定
- rolling budgetのlaunch countは同じ`runtime_instance_id/pool_name/runner_id`かつwindow内`supressed=0` decision row数
- `reason`はfailure/restart decision診断用non-empty code。抑止理由は少なくとも`policy_never|restart_limit_exceeded`
- Parent Runtimeが変わればbudget scopeも変わる

```sql
CREATE INDEX ix_runner_restart_window
ON runner_restarts(runtime_instance_id,pool_name,runner_id,created_at DESC);
```

`exited_runner_instance_id`が存在するfailureでは同じinstanceのdecision重複をRepositoryで拒否する。Spawn前failure等instance IDを持てない場合もSupervisorの一意failure fenceでexactly onceを保証する。MVPでは追加tableは作らない。

## 19. `external_tasks`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
status TEXT NOT NULL
lease_seconds REAL NOT NULL CHECK(lease_seconds>0)
on_lease_expiry TEXT NOT NULL CHECK(on_lease_expiry IN('requeue','fail'))
available_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(attempt_id)
```

`lease_seconds`はService validationでfinite positiveを保証する。

Status=`available|leased|completed|cancelled`。

- completed -> completed_at non-NULL
- cancelled -> cancelled_at non-NULL
- available|leased -> terminal timestamps NULL

Requeueでlease config変更無し。

```sql
CREATE INDEX ix_external_task_available
ON external_tasks(status,available_at,id)
WHERE status='available';
```

## 20. `external_leases`

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL REFERENCES external_tasks(id) ON DELETE NO ACTION
claimant_key TEXT NOT NULL CHECK(length(claimant_key)>0)
status TEXT NOT NULL
claimed_at TEXT NOT NULL
expires_at TEXT NOT NULL
released_at TEXT NULL
invalidated_at TEXT NULL
created_at TEXT NOT NULL
```

`claimant_key` exact semantics=`12 §2.1 actor_principal_key`。Client requestから任意文字列を保存せず、Service caller ActorContext/AccessScopeからCoreが算出する。

Status=`active|expired|released|invalidated`。

```sql
CREATE UNIQUE INDEX uq_external_active_lease
ON external_leases(task_id)
WHERE status='active';
CREATE INDEX ix_external_lease_due
ON external_leases(expires_at)
WHERE status='active';
```

Heartbeat/renew/transfer column無し。`now >= expires_at`ならLeaseはsubmit不可でexpiry処理対象。Submitはactive Lease IDに加えてcurrent caller `actor_principal_key == claimant_key`を必須確認する。

## 21. `human_reviews`

```text
id TEXT PRIMARY KEY
workflow_run_id TEXT NOT NULL REFERENCES workflow_runs(id) ON DELETE NO ACTION
job_run_id TEXT NOT NULL REFERENCES job_runs(id) ON DELETE NO ACTION
attempt_id TEXT NOT NULL REFERENCES job_attempts(id) ON DELETE NO ACTION
status TEXT NOT NULL
outcome TEXT NULL
comment TEXT NULL
actor_json TEXT NULL
created_at TEXT NOT NULL
completed_at TEXT NULL
cancelled_at TEXT NULL
UNIQUE(attempt_id)
```

Status=`pending|completed|cancelled`、Outcome=`NULL|approve|reject`。

- pending -> outcome/completed_at/cancelled_at NULL
- completed -> outcome non-NULL/completed_at non-NULL
- cancelled -> outcome NULL/cancelled_at non-NULL
- completed outcome rewrite不可

```sql
CREATE INDEX ix_reviews_list
ON human_reviews(status,created_at ASC,id ASC);
```

## 22. `idempotency_records`

Completed resultだけ。`reserved` row無し。

```text
scope TEXT NOT NULL
operation TEXT NOT NULL
request_key TEXT NOT NULL
request_hash TEXT NOT NULL
result_json TEXT NOT NULL
adapter_meta_json TEXT NULL
created_at TEXT NOT NULL
expires_at TEXT NOT NULL
PRIMARY KEY(scope,operation,request_key)
```

`request_hash`=SHA-256(canonical-json-v1(Service request excluding request_id/transport-only fields))。

`actor_principal_key` exact definition=`12 §2.1`でAccessScopeを既に含む。

Idempotency `scope` source:

```text
namespace
operation_resource_key
actor_principal_key
```

をcanonical-json-v1でserialize/hashしたopaque key。AccessScopeを別fieldとして二重追加しない。

Flow=optional fast read -> prepare -> `BEGIN IMMEDIATE` -> key/hash再確認 -> domain state再確認 -> side effect + result row commit。

TTL=System `idempotency_ttl_hours`, default24h。`now >= expires_at`でexpired扱いし同keyを新requestとしてreplace可能。

```sql
CREATE INDEX ix_idempotency_expiry
ON idempotency_records(expires_at);
```

## 23. Concurrency / priority transaction

Concurrency dedicated table無し。

### 23.1 Scope / holder

Concurrency scope key:

```text
(workflow_id, concurrency_group)
```

Workflow Definition単位。同じgroup文字列でも別`workflow_id`なら競合しない。GroupはBINARY exact comparison。

Canonical holder/waiter:

```text
holder = status='running'
      OR (status='paused' AND wait_reason IS NULL)
waiter = status='queued' AND wait_reason='concurrency' AND concurrency_queued_at IS NOT NULL
paused waiter = status='paused' AND wait_reason='concurrency' AND concurrency_queued_at IS NOT NULL
```

Concurrency groupがNULLのRunはslot count対象外。

Candidate Runがslot取得を試みるとき、同じscopeのholderだけをcountする。

### 23.2 Capacity

Candidateの`concurrency_max_runs`と既存holderの`concurrency_max_runs`が異なり得るため、admission時のeffective capacity:

```text
min(candidate.max_runs, all active holders' max_runs)
```

holderが0件ならcandidate.max_runs。

`holder_count < effective_capacity`ならadmit。それ以外はcandidate自身の`on-limit=queue|reject`を適用。

### 23.3 Waiting order / admission timestamp

Waiting Run=`status=queued, wait_reason=concurrency, concurrency_queued_at non-NULL`。

Release時:

```text
priority DESC
concurrency_queued_at ASC
id ASC
```

`created_at`はwaiting orderへ使わない。特にManual Retryで古いRunをreopenした場合、reopen時に新しい`concurrency_queued_at`を与えるため既存waiterへ割り込まない。

先頭から順に現在のholder集合でcapacityを再計算し、admitできない先頭waiterがあれば後続を追い越させない。Paused waiterはResumeするまで候補外で、元のqueue timestampを保持する。

Slot取得commitで:

```text
status=running
wait_reason=NULL
concurrency_queued_at=NULL
if started_at IS NULL: started_at=current admission time
```

`started_at`初回設定と`workflow_started` Eventを同一transactionにする。

Workflow start/slot release/Manual Retry reopenは`BEGIN IMMEDIATE`で判定する。

Root `wf_priority_update` は1 transactionでroot + all non-terminal descendant Child Run (`root_workflow_run_id=root.id`) priorityを同値へ更新する。Preempt無し。

## 24. Result Reuse persistence

Successful Attempt terminal時=`reuse_context_json/reuse_key/reuse_eligible`。

Manual Retry successful descendantはcurrent if/expected Input/Artifact/validationを再計算。Dynamic successful expansionはexpansion digest exact check。

Reusable executor identityには:

```text
Child Definition
Child Action/Validator versions
Child System workflow defaults
Child effective settings
Child retention policy
```

を含む。

## 25. Retention / delete order

Policy=`workflow_runs.retention_policy_json`。

- Run completed_at、non-terminal delete無し
- Log close time、running delete無し
- normal Event created_at、retention audit除外
- Artifact data/metadataは`09`の期限優先規則
- Output Payload run-historyと共にdelete
- Cross-run ArtifactRef pin無し
- External Artifact実体delete無し

Child subtreeは`call_depth DESC`で先に削除。Parent run-history expiryはChild subtree実効上限。

1 Run代表順:

1. descendant Child subtree
2. external_leases
3. human_reviews
4. execution log
5. Artifact data -> Artifact metadata row
6. state
7. normal events
8. job_steps
9. external_tasks
10. dynamic_expansions
11. job_attempts
12. reusable_bindings
13. job_runs
14. Workflow Output blob
15. workflow_runs

Runner current pointerはrepair後。FK無効化禁止。

## 26. Transaction boundaries

1 DB transaction:

- Workflow Run start + System/effective snapshot + concurrency admission/queue timestamp + `started_at` initial admission semantics + optional idempotency
- root priority update + descendant propagation + idempotency
- internal claim + Attempt + pending copy + Runner ownership
- Dynamic expansion
- state current+history
- External Lease claim with canonical claimant_key + idempotency
- External submit + claimant ownership check + optional claim_next same claimant Lease + full idempotency
- Human review first-wins + idempotency
- Reusable binding + Parent Attempt transition + Child concurrency admission/queue timestamp + Child `started_at` semantics; admit/queue時だけChild Run作成、reject時はChild row無しでParent Attempt failure
- Manual Retry non-terminal: Job terminal fields clear + pending snapshot + descendant/reuse bookkeeping + idempotency
- Manual Retry reopen: concurrency reacquire + fresh queue timestamp when queued + Run/Job terminal fields clear + Workflow Output clear + pending snapshot + **started_at保持** + idempotency
- concurrency holder release/wake + admission clears queue timestamp + first admission sets started_at/workflow_started Event

Completed failure Run Manual Retryで`on-limit=reject`かつslot不可の場合はtransactionを変更せずconflictとして終了する。Queueの場合はRunをnon-terminalへreopenし`status=queued, wait_reason=concurrency, concurrency_queued_at=retry transaction time`でpending Inputを保持する。

Payload/Artifact filesystem=prepare/finalize + DB transaction + orphan cleanup。Distributed exactly-once無し。

## 27. Migration / verification

Migration=`migrations/NNN_name.sql`。

Startup: schema read -> duplicate/gap/future reject -> pending sequential -> expected version verify。

Tests use`sqlite_master`/PRAGMA for exact table/column/index/FK/check。

## 28. 受入条件

1. 全18 table exact schema
2. canonical timestamp fixed UTC format / lexical order
3. Workflow created_at/started_at/completed_at exact semantics
4. never-admitted cancelled Run may have started_at NULL
5. first admission sets started_at/workflow_started Event exactly once
6. Manual Retry preserves started_at and clears/replaces completed_at
7. Workflow queued/running/paused/completed + wait_reason/slot/concurrency_queued_at invariant
8. initial/Child/Manual-Retry queue timestamp + admission clear + pause/resume preservation
9. Dynamic template no job_runs
10. system_workflow_defaults/effective_settings exact shape
11. Root System baseline snapshot / Child copy
12. explicit/omitted runs_on stored resolved
13. static source_order=0 + job_key stable ordering
14. all-executor queued pending invariant + retry clears terminal Job fields
15. Attempt Secret bindings/input digest + secret-bound reuse_eligible=0
16. one-open-Step partial unique + start/finish metadata split
17. Dynamic expansion digest/unique
18. Reusable binding Child System baseline/settings/Retention + reject no Child row
19. Child priority=root priority + update propagation
20. State immediate nonrollback + open Step producer association
21. Artifact schema has no metadata soft-delete column
22. Runner invariants/indexes
23. runner_restarts exactly-one decision + suppressed ordinal/schedule semantics
24. External Task lease finite/config snapshot + expiry boundary
25. Lease claimant_key exact actor principal + submit ownership check
26. Human immutable
27. Idempotency request hash/scope/no-reserved/recheck/expiry boundary + canonical actor principal
28. concurrency scope=(workflow_id,group)
29. mixed max-runs conservative capacity + waiter ordering by concurrency_queued_at/paused waiter exclusion
30. Manual Retry concurrency reacquire/output clear atomicity
31. submit+claim_next atomic same claimant
32. Result Reuse identities
33. Child-first Retention
34. migration verification
35. Payload/Artifact crash consistency
