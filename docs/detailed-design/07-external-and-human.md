# 07. External / Human Executor 詳細設計

- Status: Draft v0.4
- 対象: MVP
- 上位仕様: `docs/design.md`
- 関連: `01`, `02`, `03`, `08`, `09`, `10`, `11`, `12`

## 1. 共通原則

1. External/Human JobはRunnerを占有しない。
2. `action/uses/runs-on/timeout-minutes`は禁止。
3. Secret参照は禁止。
4. Job Inputはactivation時snapshot。
5. activation時にAttempt作成。
6. Retryはnew Attempt。
7. state changeはService/Authorization/idempotency経路。

## 2. External LLM

```yaml
jobs:
  classify:
    executor: external_llm
    with:
      text: ${{ inputs.text }}
    external:
      lease-minutes: 120
      on-lease-expiry: requeue
```

Default lease=60分、expiry=`requeue`。

## 3. External activation

1 transaction:

1. final Input/continue-on-error確定
2. new Attempt
3. External Task
4. Job=`waiting_external`
5. Event

One Task / Attempt。

## 4. Task / Lease

Task:

```text
available|leased|completed|cancelled
```

Lease:

```text
active|expired|released|invalidated
```

`task_claim` candidate順:

1. Workflow priority DESC
2. Job priority DESC
3. Dynamic order rank ASC
4. source order ASC
5. ready_at ASC
6. Job Run ID

Atomic exactly-one claim。

## 5. Lease expiry

`requeue`:

- same Attempt
- Lease expired
- Task available

`fail`:

- Attempt failure `external_lease_expired`
- Retry policy

Pause中もlease clockは進む。

## 6. `task_submit`

Request:

```text
task_id
lease_id
result                 # 任意JSON-compatible value
artifacts optional     # External Reference Artifact metadataのみ
claim_next optional
request_id optional
```

Resultはobjectに限定しない。scalar/list/object/nullを許可し、optional JSON Schema / `success_if` を適用する。

### 6.1 Output persistence

Resultは`02/08`のPayloadStoreへ保存する。

- <= threshold: SQLite inline
- > threshold: filesystem blob
- sizeによるhidden failure無し

SecretはExternal Jobへ注入されない。

### 6.2 Artifact

`task_submit.artifacts` は `storage_kind=external` の参照登録だけ。URI/metadataを受け、Coreは実体fetch/uploadしない。

MVP external task protocolからmanaged ArtifactStoreへbinary uploadする機能は持たない。

### 6.3 Atomic submit

1 transaction相当のService operationで:

- current Lease validation
- result validation / PayloadStore準備
- external Artifact metadata
- Attempt/Job terminal
- Lease release / Task complete
- Event / idempotency result

Payload blobのfilesystem/DB crash consistencyは`08`。

## 7. `claim_next`

Submit成功後、別transactionで通常`task_claim`と同じselection/authorizationを使用。Next claim failureでsubmitをrollbackしない。

## 8. Pause / Cancel / Recovery

Pause:

- new claim禁止
- existing Lease submit受理
- lease expiry継続

Cancel:

- available Task cancelled
- active Lease invalidated
- Attempt/Job cancelled
- late submit reject

Recovery:

- active unexpired Lease維持
- expiredへpolicy
- terminal Attemptへnew Lease不可

## 9. Human Review

```yaml
jobs:
  approve:
    executor: human
    with:
      summary: ${{ needs.analyze.outputs }}
```

Humanは`action/uses/runs-on/success_if/external/timeout-minutes/secrets`禁止。

Activation: new Attempt + one pending Review + Job=`waiting_review`。

Outcome:

```text
approve|reject
```

Approve -> success。
Reject -> failure `human_rejected`, retryable=false default。

Human lease/timeout無し。Concurrent submitはfirst-wins。

Reviewの標準Job OutputはMVPでは `null` とする。Review metadataは`wf_review_info`から取得し、Job Outputへ自動混入しない。

## 10. Human Pause / Cancel / Retry

Pause中review submit受理。

Cancel後late submit reject。

Reject後Retryはnew Attempt + new pending Review。

## 11. Idempotency / Authorization

`task_claim/task_submit/review_submit` はoptional `request_id`。共通scope/TTLは`08/11`。

All read/write Authorization。

## 12. 受入条件

1. arbitrary JSON External result
2. large result transparent spill
3. external Artifact reference only
4. concurrent claim exactly one
5. claim/claim_next同順序
6. lease requeue/fail
7. pause/cancel/recovery
8. Human output null + review info separation
9. Human first-wins/retry
10. idempotency/authorization
