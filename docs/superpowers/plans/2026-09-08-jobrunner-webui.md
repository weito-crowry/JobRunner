# JobRunner WebUI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the MVP JobRunner WebUI defined in `docs/detailed-design/14-webui.md` as an embedded FastAPI + React control panel that uses the canonical JobRunner Service layer, provides safe operational mutations, supports coarse SSE change notification with polling fallback, and packages static assets into `jobrunner[web]`.

**Architecture:** The browser uses REST as the canonical state source, SSE only as a coarse invalidation signal, and offset-based polling for high-frequency Attempt Execution Logs. The FastAPI adapter resolves Parent-owned authentication into `ActorContext`/`AccessScope` and calls the same strict Core Service models used by MCP/Python; it never reads/writes SQLite, PayloadStore, ArtifactStore, or Log files directly. One mounted WebUI instance binds to one Runtime.

**Tech Stack:** Python >=3.10, Pydantic v2 strict models, FastAPI, SQLite, React, TypeScript, Vite, React Router, TanStack Query/Table/Virtual, Radix UI, shadcn/ui-style components, Tailwind CSS, React Flow, CodeMirror 6, Vitest, React Testing Library, MSW, Playwright.

**Spec:** `docs/detailed-design/14-webui.md` (plus canonical contracts in `docs/design.md` and `docs/detailed-design/01`–`13`).

## Global Constraints

- **Execution prerequisite:** Do not implement this plan against the current design-only repository. Before Task 1 starts, Core implementation for `01`–`13` must exist with the canonical services, persistence, auth, stores, Runner model, HTTP error model, and tests described below. If those modules do not exist, stop and implement the Core plans first; do not create Web-only duplicate domain models as a shortcut.
- Python >=3.10.
- Base package must not depend on FastAPI or frontend build tooling; Web Python dependencies belong in `jobrunner[web]`, and `[all]` includes them.
- Core Service models remain strict/no-coercion: `"50" != 50`, `"true" != true`, `true != 1`.
- HTTP query/path parsing may explicitly parse transport strings; JSON request bodies must preserve JSON types.
- REST is the canonical browser state source. SSE carries only coarse change notifications, never business payload bodies.
- Execution Log remains Attempt-level. Step selection must never imply Step-level log filtering.
- One mount = one Runtime. No Runtime selector in the SPA.
- Parent owns Authentication. Every public business read/write continues to pass `AuthorizationProvider`.
- Web Adapter must not access DB/Store/files directly. Add Service methods where WebUI needs read/stream access.
- WebUI introduces no generic Job state override, public State mutation, Runner kill/restart/scale, External Task human worker UI, Runtime config write, Workflow editor, or JobRunner-owned login.
- State-changing HTTP uses `Idempotency-Key`; body `request_id` remains absent.
- Run `queued` means Concurrency wait only. Never substitute `created_at` for null `started_at`.
- Existing persistence remains exactly the 18 canonical tables. Do not add a durable Web notification table for SSE.
- Managed Artifact data may be streamed after `artifact.read` authorization; External Reference Artifact data is not proxied/fetched by Core.
- Current design branch for this plan is `plan/webui-implementation`; implementation should use a fresh worktree/branch at execution time.

## Prerequisite Interface Gate

Before execution, verify these real Core interfaces exist (names may already be established by the Core implementation; if they differ, update this plan once to match reality rather than adding aliases):

```python
WorkflowDefinitionService
WorkflowRunService
InputService
OutputService
WorkflowStateService
ExternalTaskService
HumanReviewService
ArtifactService
LogService
EventService
RunnerService
AuthorizationProvider
ActorContext
AccessScope
```

Verify the Core also exposes:

```text
- canonical HTTP error model: code/message/retryable/field-or-path?/details?
- 18-table SQLite persistence
- LocalArtifactStore/PayloadStore/Execution Log storage behind Service abstractions
- wf_start/pause/resume/cancel/retry/priority_update/review_submit semantics
- Run/Job/Attempt/Step read shapes from detailed design 11
- public State read/history, Event read, Runner info
```

If any of the above is absent, WebUI work is blocked because the adapter must not invent substitute semantics.

---

## File Structure to Lock Before Coding

The implementation should converge on this structure. Files may be split further if they grow, but responsibilities must not be merged across these boundaries.

```text
jobrunner/
├─ services/
│  ├─ dashboard.py              # dashboard aggregate read projection
│  ├─ web_projections.py        # Job/Attempt info + available_actions projections
│  ├─ runtime_info.py           # runtime + runner restart read projection
│  └─ change_feed.py            # coarse cross-process DB change detection
├─ artifacts/
│  └─ service.py                # add authorized managed-data stream handle
├─ logs/
│  └─ service.py                # add authorized full-log stream handle
└─ adapters/
   └─ web/
      ├─ __init__.py
      ├─ router.py              # FastAPI router composition only
      ├─ dependencies.py        # ActorContext/AccessScope resolver dependencies
      ├─ models.py              # Web transport/bootstrap/SSE-only models
      ├─ errors.py              # Core error -> HTTP response mapping
      ├─ api_routes.py          # JSON API route registration
      ├─ streaming.py           # Artifact/log StreamingResponse helpers
      ├─ sse.py                 # SSE transport + keepalive
      ├─ static.py              # SPA/index/static delivery
      └─ static/                # Vite build output packaged in wheel

webui/
├─ src/
│  ├─ app/
│  │  ├─ App.tsx
│  │  ├─ router.tsx
│  │  ├─ query-client.ts
│  │  └─ bootstrap.ts
│  ├─ api/
│  │  ├─ client.ts
│  │  ├─ errors.ts
│  │  ├─ stream.ts
│  │  ├─ query-keys.ts
│  │  └─ generated/
│  ├─ components/
│  │  ├─ ui/
│  │  ├─ layout/
│  │  ├─ status/
│  │  ├─ json/
│  │  └─ common/
│  ├─ features/
│  │  ├─ dashboard/
│  │  ├─ workflows/
│  │  ├─ runs/
│  │  ├─ jobs/
│  │  ├─ attempts/
│  │  ├─ logs/
│  │  ├─ state/
│  │  ├─ events/
│  │  ├─ reviews/
│  │  ├─ artifacts/
│  │  ├─ runners/
│  │  └─ runtime/
│  ├─ routes/
│  └─ main.tsx
├─ tests/
├─ package.json
├─ package-lock.json
├─ tsconfig.json
└─ vite.config.ts

tests/
├─ web/
│  ├─ test_web_api.py
│  ├─ test_web_auth.py
│  ├─ test_web_streaming.py
│  ├─ test_web_sse.py
│  ├─ test_web_static.py
│  └─ test_web_packaging.py
└─ e2e/
   ├─ test_webui_smoke.py
   └─ fixtures/
```

---

### Task 1: Synchronize the Normative Design Contracts Before Code

**Files:**
- Modify: `docs/design.md`
- Modify: `docs/detailed-design/11-service-api-and-mcp.md`
- Modify: `docs/detailed-design/12-security-and-secrets.md`
- Modify: `docs/detailed-design/13-testing.md`
- Reference: `docs/detailed-design/14-webui.md`

**Interfaces:**
- Consumes: approved WebUI spec in `14-webui.md`.
- Produces: one coherent normative contract for the Core/API work in later tasks.

- [ ] **Step 1: Add WebUI scope to the basic design**

Update the stale `WebUI: 画面構成のみ後続` wording in `docs/design.md` to state that MVP includes an embedded human operations WebUI, while retaining the existing non-goals: no GUI Workflow editor, no auth infrastructure, no Cron/CLI, no central multi-Runtime console.

- [ ] **Step 2: Add the canonical read projections and routes to detailed design 11**

Add these Service operations without changing existing mutation semantics:

```text
wf_dashboard_info
wf_job_info
wf_attempt_info
wf_artifact_data_read
wf_log_stream
wf_runner_restart_list
wf_runtime_info
```

Add `available_actions` to Run/Job/Review info projections. Add HTTP routes:

```text
GET /dashboard
GET /jobs/{job_run_id}
GET /attempts/{attempt_id}
GET /artifacts/{artifact_id}/data
GET /attempts/{attempt_id}/log/data
GET /runner-restarts
GET /runtime
GET /web/bootstrap
GET /stream
```

Clarify that `/web/bootstrap` and `/stream` are Web adapter control endpoints and are not MCP tools.

- [ ] **Step 3: Add Web security boundaries to detailed design 12**

Document:

```text
Parent authentication -> ActorContext/AccessScope -> Service AuthorizationProvider
artifact data stream -> artifact.read
log data stream -> same authorization as canonical log read
SSE/bootstrap -> authenticated adapter control endpoints with no business payload bodies
same-origin by default; broad CORS not enabled by JobRunner
Parent CSRF policy applies to state-changing JobRunner HTTP routes
```

- [ ] **Step 4: Add Web test requirements to detailed design 13**

Add explicit acceptance tests for:

```text
FastAPI adapter contract
strict Workflow Start browser typing
available_actions projection
artifact/log streaming auth
SSE reconnect + polling fallback
cross-process SQLite change detection
browser deep-link reload
Human Review race
large dynamic-job/log rendering
accessibility smoke
wheel contains built SPA assets
```

- [ ] **Step 5: Review the four docs for contradictions**

Run text searches for the old constraints that would now be false:

```bash
grep -R "WebUI.*後続\|Settings write\|WebUI.*画面構成" docs/design.md docs/detailed-design
```

Expected: no statement contradicts the approved `14-webui.md`; Runtime settings remain read-only in MVP.

- [ ] **Step 6: Commit**

```bash
git add docs/design.md docs/detailed-design/11-service-api-and-mcp.md docs/detailed-design/12-security-and-secrets.md docs/detailed-design/13-testing.md
git commit -m "docs: align core contracts with WebUI design"
```

---

### Task 2: Add Canonical Web Read Projections to the Service Layer

**Files:**
- Create: `jobrunner/services/dashboard.py`
- Create: `jobrunner/services/web_projections.py`
- Create: `jobrunner/services/runtime_info.py`
- Modify: existing `jobrunner/services/__init__.py`
- Modify: existing `WorkflowRunService` implementation file
- Modify: existing `HumanReviewService` implementation file
- Test: `tests/services/test_web_projections.py`

**Interfaces:**
- Consumes: existing Repository/Service domain objects and `AuthorizationProvider`.
- Produces:

```python
DashboardService.get_summary(actor, scope, failed_from=None) -> DashboardSummary
WorkflowRunService.job_info(actor, scope, request: JobInfoRequest) -> JobInfo
WorkflowRunService.attempt_info(actor, scope, request: AttemptInfoRequest) -> AttemptInfo
RunnerRuntimeInfoService.runner_restart_list(actor, scope, request) -> Page[RunnerRestartInfo]
RunnerRuntimeInfoService.runtime_info(actor, scope) -> RuntimeInfo
```

`RunInfo`, `JobInfo`, and `ReviewInfo` expose `available_actions` derived from current domain state + authorization.

- [ ] **Step 1: Write failing tests for `available_actions`**

Create tests covering at least:

```python
@pytest.mark.parametrize(
    ("status", "is_child", "allowed", "expected"),
    [
        ("running", False, True, {"pause": True, "resume": False, "cancel": True, "priority_update": True}),
        ("paused", False, True, {"pause": False, "resume": True, "cancel": True, "priority_update": True}),
        ("completed", False, True, {"pause": False, "resume": False, "cancel": False, "priority_update": False}),
        ("running", True, True, {"pause": False, "resume": False, "cancel": False, "priority_update": False}),
        ("running", False, False, {"pause": False, "resume": False, "cancel": False, "priority_update": False}),
    ],
)
def test_run_available_actions(...): ...
```

Add Job retry tests where domain retry eligibility and authorization are both required, and Review submit tests where only `pending` review is actionable.

- [ ] **Step 2: Run the focused tests and verify failure**

```bash
pytest tests/services/test_web_projections.py -q
```

Expected: FAIL because projection models/methods do not exist.

- [ ] **Step 3: Implement strict projection models and action calculation**

Use strict Pydantic models for JSON-compatible projections. Keep authorization checks inside Service code; do not expose authorization credentials or reasons containing sensitive policy data.

```python
class RunAvailableActions(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True, frozen=True)
    pause: bool
    resume: bool
    cancel: bool
    priority_update: bool

class JobAvailableActions(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True, frozen=True)
    retry: bool

class ReviewAvailableActions(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True, frozen=True)
    submit: bool
```

- [ ] **Step 4: Add `job_info` and `attempt_info` using existing canonical nested shapes**

Do not define a Web-specific Job/Attempt semantic model. Reuse the exact Job/Attempt/Step fields already returned inside `wf_run_info`, adding only direct selectors and `available_actions`.

- [ ] **Step 5: Add Dashboard and Runtime/Restart projections**

Dashboard queries must apply the same AccessScope filtering as canonical list APIs. Runtime config values are observation-only and include their source only when the Core can determine it without guessing.

- [ ] **Step 6: Run focused + existing Service tests**

```bash
pytest tests/services/test_web_projections.py tests/services -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/services tests/services/test_web_projections.py
git commit -m "feat: add WebUI read projections"
```

---

### Task 3: Add Authorized Artifact and Log Stream Handles

**Files:**
- Modify: existing `jobrunner/artifacts/service.py`
- Modify: existing LogService implementation file
- Test: `tests/services/test_artifact_stream.py`
- Test: `tests/services/test_log_stream.py`

**Interfaces:**
- Consumes: canonical Artifact metadata, ArtifactStore, Execution Log storage, ActorContext/AccessScope.
- Produces:

```python
@dataclass(frozen=True)
class ArtifactDataHandle:
    artifact_id: str
    name: str
    media_type: str | None
    size_bytes: int
    digest: str
    stream: BinaryIO

@dataclass(frozen=True)
class LogDataHandle:
    attempt_id: str
    size_bytes: int
    stream: BinaryIO
```

Service methods:

```python
ArtifactService.open_data(actor, scope, artifact_id: str) -> ArtifactDataHandle
LogService.open_stream(actor, scope, attempt_id: str) -> LogDataHandle
```

- [ ] **Step 1: Write negative tests first**

Cover:

```text
external Artifact -> artifact_data_unavailable
metadata deleted -> not_found
managed data deleted -> artifact_data_unavailable
artifact.read forbidden -> artifact_access_forbidden/forbidden per existing canonical mapping
Store object missing -> artifact_data_unavailable or canonical integrity error
log deleted -> log_data_unavailable
log read forbidden -> forbidden
```

- [ ] **Step 2: Run tests and verify failure**

```bash
pytest tests/services/test_artifact_stream.py tests/services/test_log_stream.py -q
```

- [ ] **Step 3: Implement Service-owned stream opening**

The Service must perform existence, storage-kind, retention/deletion, authorization, and Store integrity checks before returning a handle. Do not expose filesystem/store keys through public response models.

- [ ] **Step 4: Verify existing ArtifactRef cross-run authorization remains unchanged**

Run the existing Artifact authorization/reuse tests plus the new stream tests.

```bash
pytest tests/services/test_artifact_stream.py tests/services/test_log_stream.py tests -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add jobrunner/artifacts jobrunner tests/services/test_artifact_stream.py tests/services/test_log_stream.py
git commit -m "feat: add authorized artifact and log streams"
```

---

### Task 4: Add the Coarse Web Change Feed Service

**Files:**
- Create: `jobrunner/services/change_feed.py`
- Test: `tests/services/test_change_feed.py`
- Test: `tests/e2e/test_change_feed_process.py`

**Interfaces:**
- Consumes: dedicated SQLite read connection to the same DB file.
- Produces:

```python
@dataclass(frozen=True)
class ChangeNotice:
    sequence: int
    occurred_at: str

class WebChangeFeedService:
    def current_version(self) -> int: ...
    async def wait_for_change(self, previous_version: int, timeout_seconds: float) -> ChangeNotice | None: ...
```

Implementation rule: use a dedicated read connection and `PRAGMA data_version`; no nineteenth persistence table and no domain Event duplication.

- [ ] **Step 1: Write a unit test around a dedicated change-feed connection**

The test must open connection A for the feed and connection B for a write, commit on B, and assert A observes a changed `PRAGMA data_version`.

- [ ] **Step 2: Run the test and verify failure**

```bash
pytest tests/services/test_change_feed.py -q
```

- [ ] **Step 3: Implement `WebChangeFeedService`**

Poll at a bounded interval, default 250 ms, and coalesce multiple DB commits into one returned notice. Use an in-memory monotonically increasing `sequence`; it is not a durable audit sequence.

- [ ] **Step 4: Add a real child-process test**

Spawn a child process that commits a normal JobRunner DB update and verify the Parent process feed detects it. This test is required because Runner writes occur in separate processes.

- [ ] **Step 5: Run tests**

```bash
pytest tests/services/test_change_feed.py tests/e2e/test_change_feed_process.py -q
```

Expected: PASS on Windows spawn and Linux.

- [ ] **Step 6: Commit**

```bash
git add jobrunner/services/change_feed.py tests/services/test_change_feed.py tests/e2e/test_change_feed_process.py
git commit -m "feat: add WebUI change feed"
```

---

### Task 5: Build the FastAPI Web Adapter Foundation

**Files:**
- Create: `jobrunner/adapters/web/__init__.py`
- Create: `jobrunner/adapters/web/router.py`
- Create: `jobrunner/adapters/web/dependencies.py`
- Create: `jobrunner/adapters/web/models.py`
- Create: `jobrunner/adapters/web/errors.py`
- Create: `jobrunner/adapters/web/api_routes.py`
- Modify: `pyproject.toml`
- Test: `tests/web/test_web_api.py`
- Test: `tests/web/test_web_auth.py`

**Interfaces:**
- Consumes: canonical Service facade/object, Parent actor resolver, Parent scope resolver.
- Produces:

```python
ActorResolver = Callable[[Request], ActorContext]
AccessScopeResolver = Callable[[Request], AccessScope]

def create_jobrunner_web_router(
    service: JobRunnerService,
    actor_resolver: ActorResolver,
    access_scope_resolver: AccessScopeResolver,
    *,
    ui_prefix: str = "/jobrunner",
    api_prefix: str = "/api/jobrunner/v1",
    display_name: str | None = None,
) -> APIRouter:
    ...
```

- [ ] **Step 1: Add `[web]` optional Python dependencies and failing import test**

`pyproject.toml` must keep base install Web-free while adding FastAPI and its required runtime dependencies to `[web]`; `[all]` includes `[web]` and `[mcp]` dependencies.

Write a test that imports `jobrunner.adapters.web` only when Web extras are installed.

- [ ] **Step 2: Write API contract tests before route implementation**

Use FastAPI `TestClient`/httpx and a fake canonical Service object. Cover at minimum:

```text
GET /dashboard
GET /jobs/{id}
GET /attempts/{id}
GET /runner-restarts
GET /runtime
POST /workflow-runs
POST /workflow-runs/{id}/pause
POST /workflow-runs/{id}/resume
POST /workflow-runs/{id}/cancel
PATCH /workflow-runs/{id}
POST /workflow-runs/{id}/jobs/{job_id}/retry
POST /reviews/{review_id}/submit
```

Assert `Idempotency-Key` maps to Service `request_id`, JSON body types are not coerced, and adapter never returns HTTP 422 for canonical validation failures.

- [ ] **Step 3: Run focused tests and verify failure**

```bash
pytest tests/web/test_web_api.py tests/web/test_web_auth.py -q
```

- [ ] **Step 4: Implement resolver dependencies and Core error mapping**

Map canonical status codes exactly:

```text
400 validation/domain
401 unauthenticated
403 forbidden
404 not_found
409 conflict/state/idempotency/concurrency/lease
413 explicit adapter size policy
500 internal_error
```

- [ ] **Step 5: Implement JSON routes as thin Service calls**

`api_routes.py` may parse HTTP transport strings but must not contain Repository/SQLite/Store imports. Add a structural test that fails if these modules are imported from `jobrunner.adapters.web`.

- [ ] **Step 6: Run tests**

```bash
pytest tests/web/test_web_api.py tests/web/test_web_auth.py -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add pyproject.toml jobrunner/adapters/web tests/web/test_web_api.py tests/web/test_web_auth.py
git commit -m "feat: add FastAPI Web adapter"
```

---

### Task 6: Add Web Bootstrap, Artifact/Log Streaming, and SSE Transport

**Files:**
- Create: `jobrunner/adapters/web/streaming.py`
- Create: `jobrunner/adapters/web/sse.py`
- Modify: `jobrunner/adapters/web/models.py`
- Modify: `jobrunner/adapters/web/router.py`
- Test: `tests/web/test_web_streaming.py`
- Test: `tests/web/test_web_sse.py`

**Interfaces:**
- Consumes: `ArtifactService.open_data`, `LogService.open_stream`, `WebChangeFeedService`.
- Produces HTTP:

```text
GET /web/bootstrap
GET /artifacts/{artifact_id}/data
GET /attempts/{attempt_id}/log/data
GET /stream
```

Bootstrap shape:

```json
{
  "runtime_instance_id": "runtime_...",
  "system_namespace": "novel",
  "display_name": "NovelProduction",
  "api_version": "v1",
  "webui_version": "...",
  "features": {
    "workflow_start": true,
    "run_control": true,
    "review": true,
    "artifact_data_read": true,
    "sse": true
  }
}
```

SSE data frame contains only:

```json
{"type":"runtime.changed","sequence":1,"occurred_at":"..."}
```

plus `stream.ready` on connect/reconnect and comment keepalives.

- [ ] **Step 1: Write streaming authorization/retention tests**

Assert the adapter receives only the authorized Service stream handle, sets safe `Content-Disposition`, does not reveal store paths, and rejects External Reference Artifact data.

- [ ] **Step 2: Write SSE tests**

Cover:

```text
authentication required
stream.ready emitted on connect
runtime.changed emitted after ChangeFeed notice
business payload bodies absent
keepalive comments emitted
client disconnect cancels waiter cleanly
```

- [ ] **Step 3: Run tests and verify failure**

```bash
pytest tests/web/test_web_streaming.py tests/web/test_web_sse.py -q
```

- [ ] **Step 4: Implement streaming helpers**

Sanitize download filenames for HTTP headers. Never construct Store/log filesystem paths in adapter code.

- [ ] **Step 5: Implement SSE as coarse invalidation**

Use a small async loop around `WebChangeFeedService.wait_for_change`. Do not implement durable replay, resource-specific IDs, or Event-table mirroring.

- [ ] **Step 6: Run tests**

```bash
pytest tests/web/test_web_streaming.py tests/web/test_web_sse.py -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/adapters/web tests/web/test_web_streaming.py tests/web/test_web_sse.py
git commit -m "feat: add Web streaming and SSE"
```

---

### Task 7: Scaffold the React/TypeScript Workspace and Generated API Client

**Files:**
- Create: `webui/package.json`
- Create: `webui/package-lock.json`
- Create: `webui/tsconfig.json`
- Create: `webui/vite.config.ts`
- Create: `webui/src/main.tsx`
- Create: `webui/src/app/App.tsx`
- Create: `webui/src/app/query-client.ts`
- Create: `webui/src/api/client.ts`
- Create: `webui/src/api/errors.ts`
- Create: `webui/src/api/query-keys.ts`
- Create: `webui/src/api/generated/`
- Create: `webui/tests/setup.ts`

**Interfaces:**
- Consumes: FastAPI OpenAPI document from Task 5/6.
- Produces: typed browser API client and QueryClient, with no handwritten duplicate of canonical response shapes.

- [ ] **Step 1: Initialize package and install dependencies**

From `webui/`:

```bash
npm init -y
npm install react react-dom react-router-dom @tanstack/react-query @tanstack/react-table @tanstack/react-virtual @radix-ui/react-dialog @radix-ui/react-tooltip @radix-ui/react-tabs @radix-ui/react-dropdown-menu @radix-ui/react-popover @xyflow/react @uiw/react-codemirror
npm install -D typescript vite @vitejs/plugin-react vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event msw openapi-typescript tailwindcss playwright axe-core
```

Commit the generated lockfile; later installs use `npm ci`.

- [ ] **Step 2: Add scripts**

`package.json` scripts must include:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "test": "vitest run",
    "test:watch": "vitest",
    "api:generate": "openapi-typescript ../build/openapi.json -o src/api/generated/schema.d.ts"
  }
}
```

- [ ] **Step 3: Write failing client tests**

Test that `ApiClient`:

```text
uses configurable API base
uses credentials: same-origin
parses canonical error body without flattening code/details
adds Idempotency-Key only when supplied
propagates AbortSignal
```

- [ ] **Step 4: Implement `ApiClient`**

Define a single request entry point; feature modules must not call raw `fetch` for normal JSON endpoints.

```ts
export type ApiErrorBody = {
  code: string;
  message: string;
  retryable: boolean;
  field?: string;
  path?: string;
  details?: unknown;
};
```

- [ ] **Step 5: Generate types from FastAPI OpenAPI and assert clean drift**

Add a Python build helper or test fixture that writes OpenAPI to `build/openapi.json`, run `npm run api:generate`, then verify `git diff --exit-code webui/src/api/generated` after regeneration.

- [ ] **Step 6: Run frontend tests/build**

```bash
cd webui
npm test
npm run build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add webui
git commit -m "feat: scaffold typed WebUI client"
```

---

### Task 8: Implement SPA Bootstrap, Routing, Navigation, and Common State Components

**Files:**
- Create: `webui/src/app/bootstrap.ts`
- Create: `webui/src/app/router.tsx`
- Create: `webui/src/components/layout/AppShell.tsx`
- Create: `webui/src/components/layout/Sidebar.tsx`
- Create: `webui/src/components/status/StatusBadge.tsx`
- Create: `webui/src/components/common/ResourceId.tsx`
- Create: `webui/src/components/common/PageState.tsx`
- Create: `webui/src/routes/NotFoundRoute.tsx`
- Test: `webui/src/app/router.test.tsx`
- Test: `webui/src/components/status/StatusBadge.test.tsx`

**Interfaces:**
- Consumes: `/web/bootstrap`, React Router, TanStack Query.
- Produces canonical SPA routes and common Loading/Error/Empty handling.

- [ ] **Step 1: Write route/deep-link tests**

Test all canonical suffixes:

```text
/
/workflows
/workflows/:workflowId
/workflows/:workflowId/start
/runs
/runs/:runId
/runs/:runId/state
/runs/:runId/events
/runs/:runId/jobs/:jobId
/runs/:runId/jobs/:jobId/attempts/:attemptId
/reviews
/reviews/:reviewId
/artifacts/:artifactId
/runners
/runtime
```

- [ ] **Step 2: Write status-format tests**

Verify status and conclusion remain distinct, including:

```text
completed + success
completed + failure
completed + cancelled
completed + skipped
completed + blocked
queued + null conclusion
```

- [ ] **Step 3: Implement bootstrap and app shell**

Bootstrap reads `base href`, API base metadata, and `/web/bootstrap`; no hardcoded `/jobrunner` assumption.

- [ ] **Step 4: Implement connection indicator state**

Common shell exposes only:

```text
Live
Polling
Disconnected
```

Do not interpret SSE loss as disconnected when REST polling succeeds.

- [ ] **Step 5: Run tests/build**

```bash
cd webui
npm test
npm run build
```

- [ ] **Step 6: Commit**

```bash
git add webui/src
git commit -m "feat: add WebUI shell and routes"
```

---

### Task 9: Implement Workflow Browse and Strict Workflow Start

**Files:**
- Create: `webui/src/features/workflows/api.ts`
- Create: `webui/src/features/workflows/WorkflowList.tsx`
- Create: `webui/src/features/workflows/WorkflowDetail.tsx`
- Create: `webui/src/features/workflows/WorkflowDag.tsx`
- Create: `webui/src/features/workflows/WorkflowStart.tsx`
- Create: `webui/src/features/workflows/SchemaForm.tsx`
- Create: `webui/src/components/json/JsonEditor.tsx`
- Test: `webui/src/features/workflows/WorkflowStart.test.tsx`
- Test: `webui/src/features/workflows/SchemaForm.test.tsx`

**Interfaces:**
- Consumes: Definition list/info and `POST /workflow-runs`.
- Produces: read-only Definition/YAML/DAG plus typed Form/JSON Start editor.

- [ ] **Step 1: Write strict-type form tests**

At minimum:

```ts
expect(submitted.integerValue).toBe(12);
expect(submitted.stringValue).toBe("12");
expect(submitted.booleanValue).toBe(true);
expect(submitted.nullableValue).toBeNull();
```

Add negative cases ensuring `"12"` is not silently accepted for integer and `"true"` is not silently accepted for boolean in JSON mode.

- [ ] **Step 2: Write Form <-> JSON switching tests**

Invalid JSON/schema values must remain untouched and block unsafe switch; the UI may show an error but must not coerce/repair the value.

- [ ] **Step 3: Implement Workflow list/detail**

Definition source invalid/unavailable must show an error state, not stale cache represented as current source. DAG is read-only logical dependency display and must not claim scheduler execution order.

- [ ] **Step 4: Implement Start mutation with idempotency**

Generate one UUID when submit begins, reuse it for transport retry of the same unresolved request, and generate a new UUID only after the user changes/submits a new logical request.

- [ ] **Step 5: Verify successful start navigation**

On `201`, navigate to `/runs/:runId`; if `started_at` is null, show it as absent rather than deriving from `created_at`.

- [ ] **Step 6: Run tests/build**

```bash
cd webui
npm test -- WorkflowStart SchemaForm
npm run build
```

- [ ] **Step 7: Commit**

```bash
git add webui/src/features/workflows webui/src/components/json
git commit -m "feat: add workflow browse and start UI"
```

---

### Task 10: Implement Run, Job, Attempt, and Step Inspection/Control

**Files:**
- Create: `webui/src/features/runs/api.ts`
- Create: `webui/src/features/runs/RunList.tsx`
- Create: `webui/src/features/runs/RunDetail.tsx`
- Create: `webui/src/features/runs/RunActions.tsx`
- Create: `webui/src/features/runs/RunJobsPanel.tsx`
- Create: `webui/src/features/jobs/JobDetail.tsx`
- Create: `webui/src/features/jobs/RetryAction.tsx`
- Create: `webui/src/features/attempts/AttemptDetail.tsx`
- Create: `webui/src/features/attempts/StepTimeline.tsx`
- Test: `webui/src/features/runs/RunActions.test.tsx`
- Test: `webui/src/features/jobs/RetryAction.test.tsx`
- Test: `webui/src/features/attempts/StepTimeline.test.tsx`

**Interfaces:**
- Consumes: Run list/info, direct Job/Attempt info, `available_actions`, pause/resume/cancel/priority/retry mutations.
- Produces: primary operations UI.

- [ ] **Step 1: Write `available_actions` rendering tests**

Assert the UI does not reimplement domain eligibility. Buttons render from the Service projection. Mutation success triggers refetch; 409 refetches and displays canonical error.

- [ ] **Step 2: Write Child Run control tests**

A Child Run response with all direct controls unavailable must show `Controlled by parent workflow run` and no direct Pause/Resume/Cancel/Priority actions.

- [ ] **Step 3: Implement Run list filters in URL search params**

Support exactly the canonical filters from design 11. Preserve them over reload/share.

- [ ] **Step 4: Implement Run detail and Dynamic groups**

Dynamic virtual groups are grouping rows only; they never get Attempt/Log links. Generated concrete Jobs use normal Job navigation.

- [ ] **Step 5: Implement mutations**

```text
Pause/Resume -> immediate request, no destructive confirm
Cancel -> confirm + optional reason
Priority -> signed64 input, no arbitrary UI min/max
Retry -> confirm; no Input override
```

Use fresh `Idempotency-Key` per logical mutation and reuse only for unresolved transport retry.

- [ ] **Step 6: Implement Attempt/Step detail**

Show only persisted Steps. Never synthesize future Steps from Workflow Definition; never show queued/skipped scheduling state as Step state.

- [ ] **Step 7: Run tests/build**

```bash
cd webui
npm test -- RunActions RetryAction StepTimeline
npm run build
```

- [ ] **Step 8: Commit**

```bash
git add webui/src/features/runs webui/src/features/jobs webui/src/features/attempts
git commit -m "feat: add run job and attempt operations UI"
```

---

### Task 11: Implement Input/Output, State, and Event Inspection

**Files:**
- Create: `webui/src/components/json/JsonViewer.tsx`
- Create: `webui/src/features/state/StatePage.tsx`
- Create: `webui/src/features/state/StateHistory.tsx`
- Create: `webui/src/features/events/EventTimeline.tsx`
- Create: `webui/src/features/runs/InputOutputPanel.tsx`
- Test: `webui/src/features/state/StatePage.test.tsx`
- Test: `webui/src/features/events/EventTimeline.test.tsx`

**Interfaces:**
- Consumes: canonical `*-input-info/read`, `*-output-info/read`, State list/read/history, Event list.
- Produces: read-only inspection UI with lazy large-payload loading.

- [ ] **Step 1: Write metadata-before-body tests**

Input/Output page must first load info metadata and only fetch body when the user opens the body view or explicitly requests it. `available=false` is not an error.

- [ ] **Step 2: Write State read-only tests**

Verify no State set/delete controls exist. History initially requests `include_values=false`; values are fetched only when requested.

- [ ] **Step 3: Implement JSON tree/raw viewer**

Render untrusted strings as text, never HTML. Support copy and expand/collapse. JMESPath filtering must use canonical server `select` when available rather than a second browser interpretation that could diverge.

- [ ] **Step 4: Implement Event timeline**

Always retain the raw machine `event_type` in the UI even if a friendly summary is shown. Payload is collapsible and read-only.

- [ ] **Step 5: Run tests/build**

```bash
cd webui
npm test -- StatePage EventTimeline
npm run build
```

- [ ] **Step 6: Commit**

```bash
git add webui/src/components/json webui/src/features/state webui/src/features/events webui/src/features/runs/InputOutputPanel.tsx
git commit -m "feat: add state event and payload inspection"
```

---

### Task 12: Implement Human Reviews, Artifacts, Runners, Runtime, and Dashboard

**Files:**
- Create: `webui/src/features/reviews/ReviewList.tsx`
- Create: `webui/src/features/reviews/ReviewDetail.tsx`
- Create: `webui/src/features/artifacts/ArtifactDetail.tsx`
- Create: `webui/src/features/runners/RunnerPage.tsx`
- Create: `webui/src/features/runtime/RuntimePage.tsx`
- Create: `webui/src/features/dashboard/DashboardPage.tsx`
- Test: `webui/src/features/reviews/ReviewDetail.test.tsx`
- Test: `webui/src/features/artifacts/ArtifactDetail.test.tsx`
- Test: `webui/src/features/dashboard/DashboardPage.test.tsx`

**Interfaces:**
- Consumes: Review list/info/submit, Artifact info/data, Runner info/restart list, Runtime info, Dashboard summary.
- Produces: remaining top-level operator flows.

- [ ] **Step 1: Write Human Review race tests**

Default queue uses canonical Review status `pending` and oldest-first ordering. Approve/Reject require confirmation. On 409 because another actor already submitted, refetch and display terminal outcome; preserve the unsent local comment until the user leaves the page.

- [ ] **Step 2: Implement Artifact detail**

Managed Artifact shows metadata + Download. External Reference shows metadata/URI only and explicitly states JobRunner does not store managed data for it; do not add proxy download.

- [ ] **Step 3: Implement Runner and restart history**

Read-only. Show busy/idle/lost, heartbeat, current Job, restart decisions, and suppressed decisions. Do not render restart/kill/scale controls.

- [ ] **Step 4: Implement Runtime read-only page**

Render effective/current observable values and source metadata when available. No Save/Edit controls.

- [ ] **Step 5: Implement Dashboard cards/attention navigation**

Cards navigate to filtered list pages. Do not introduce a second severity domain model; use source states such as lost, restart suppressed, failed Run, pending Review.

- [ ] **Step 6: Run tests/build**

```bash
cd webui
npm test -- ReviewDetail ArtifactDetail DashboardPage
npm run build
```

- [ ] **Step 7: Commit**

```bash
git add webui/src/features/reviews webui/src/features/artifacts webui/src/features/runners webui/src/features/runtime webui/src/features/dashboard
git commit -m "feat: add review artifact runner and dashboard UI"
```

---

### Task 13: Implement SSE Invalidation, Polling Fallback, and Attempt Log Follow

**Files:**
- Create: `webui/src/api/stream.ts`
- Create: `webui/src/features/logs/useAttemptLog.ts`
- Create: `webui/src/features/logs/ExecutionLog.tsx`
- Modify: `webui/src/app/App.tsx`
- Modify: `webui/src/app/query-client.ts`
- Test: `webui/src/api/stream.test.ts`
- Test: `webui/src/features/logs/ExecutionLog.test.tsx`

**Interfaces:**
- Consumes: `GET /stream`, `GET /attempts/{id}/log` offset API, `/log/data` download.
- Produces: Live/Polling/Disconnected connection state, active-query invalidation, incremental log follow.

- [ ] **Step 1: Write SSE parser/reconnect tests**

Use fetch streaming rather than relying on native `EventSource`, so request cancellation/credentials remain under the shared client policy. Test fragmented SSE frames, comment keepalives, `stream.ready`, `runtime.changed`, connection close, and reconnect backoff.

- [ ] **Step 2: Implement active-query invalidation**

On `runtime.changed`, invalidate/refetch only active mounted queries. Do not refetch every cached historical resource.

- [ ] **Step 3: Implement polling fallback**

Use approximate intervals from the spec:

```text
active Run detail: 3s
Attempt log: 1-2s
Reviews/Dashboard/Runners: 10s
completed Run: stop automatic polling
```

Background tabs may reduce cadence. When SSE recovers, stop duplicate fallback polling except the dedicated Attempt log offset loop.

- [ ] **Step 4: Write UTF-8 log offset tests**

Frontend must use only backend `next_offset_bytes`; never derive byte offset from JavaScript string length.

- [ ] **Step 5: Implement Execution Log follow**

Initial view may use `tail_lines=500`. Follow uses returned `next_offset_bytes`. Auto-scroll only while user is at tail; scrolling upward pauses follow and shows a `new lines` indicator.

- [ ] **Step 6: Run tests/build**

```bash
cd webui
npm test -- stream ExecutionLog
npm run build
```

- [ ] **Step 7: Commit**

```bash
git add webui/src/api/stream.ts webui/src/features/logs webui/src/app
git commit -m "feat: add WebUI realtime updates and log follow"
```

---

### Task 14: Add SPA Static Serving, Multi-Mount Base Path, and Wheel Packaging

**Files:**
- Create: `jobrunner/adapters/web/static.py`
- Modify: `jobrunner/adapters/web/router.py`
- Modify: `webui/vite.config.ts`
- Modify: `pyproject.toml`
- Create/Modify: package build script or Make/CI target already used by repository
- Test: `tests/web/test_web_static.py`
- Test: `tests/web/test_web_packaging.py`

**Interfaces:**
- Consumes: Vite build output.
- Produces: package-contained SPA assets served from arbitrary mount prefix without Node at runtime.

- [ ] **Step 1: Write static route precedence tests**

Assert:

```text
/api/... -> API
existing hashed asset -> asset
SPA route under UI prefix -> index.html
missing .js/.css asset -> 404, never index.html
```

- [ ] **Step 2: Write multi-mount base tests**

Mount the same router/bundle under `/novel/jobrunner/` and `/fx/jobrunner/` in separate test apps and verify generated `<base href>` and API metadata are correct without rebuilding assets.

- [ ] **Step 3: Implement static serving/cache policy**

Hashed Vite assets:

```text
Cache-Control: public, max-age=31536000, immutable
```

`index.html`:

```text
Cache-Control: no-cache
```

Do not inject executable inline scripts for base configuration.

- [ ] **Step 4: Add package build integration**

A release build performs:

```bash
cd webui
npm ci
npm test
npm run build
```

and copies output into `jobrunner/adapters/web/static/` before wheel construction.

- [ ] **Step 5: Add wheel-content test**

Build a wheel in a clean temp directory and assert `index.html` plus hashed assets are present. Install that wheel into a clean venv and assert `import jobrunner.adapters.web` succeeds without Node/npm installed.

- [ ] **Step 6: Run tests**

```bash
pytest tests/web/test_web_static.py tests/web/test_web_packaging.py -q
```

- [ ] **Step 7: Commit**

```bash
git add jobrunner/adapters/web webui/vite.config.ts pyproject.toml tests/web
git commit -m "feat: package and serve the WebUI SPA"
```

---

### Task 15: Security Hardening and Browser E2E

**Files:**
- Create/Modify: `tests/e2e/fixtures/`
- Create: `tests/e2e/test_webui_smoke.py`
- Create: `tests/e2e/test_webui_auth.py`
- Create: `webui/e2e/workflow-run.spec.ts`
- Create: `webui/e2e/review.spec.ts`
- Create: `webui/e2e/deep-links.spec.ts`
- Create: `webui/e2e/accessibility.spec.ts`
- Modify: CI workflow files once they exist

**Interfaces:**
- Consumes: complete FastAPI adapter, real temporary SQLite Runtime, real Runner/Action Runner, built SPA.
- Produces: end-to-end acceptance evidence for the WebUI spec.

- [ ] **Step 1: Build a real E2E fixture Runtime**

Register deterministic Actions/Validators for:

```text
successful workflow
failing then retryable job
human review workflow
managed artifact output
state write/history
long-running stdout/log workflow
dynamic workflow generating many jobs
```

Use real process/SQLite behavior required by detailed design 13 rather than replacing Runner behavior with browser mocks.

- [ ] **Step 2: Write the primary operator journey**

Playwright flow:

```text
Workflow list
-> Definition detail
-> Start Form/JSON
-> POST start
-> Run detail
-> Job
-> Attempt
-> live log
-> completion
-> Output
-> Managed Artifact download
```

- [ ] **Step 3: Write retry and Human Review journeys**

Retry flow must show a new Attempt. Review flow must approve/reject only; no output rewrite.

- [ ] **Step 4: Write deep-link reload tests**

Directly navigate/reload:

```text
/runs/:id
/runs/:id/jobs/:jobId
/runs/:id/jobs/:jobId/attempts/:attemptId
/reviews/:id
/artifacts/:id
```

No route may depend on prior in-memory navigation state.

- [ ] **Step 5: Write authorization negative E2E**

Verify a user with hidden/disabled UI controls still receives 403 when calling the mutation endpoint directly. Verify unauthorized resource IDs do not leak through list responses or SSE payloads.

- [ ] **Step 6: Add XSS/secret regression cases**

Persist log/message/metadata strings containing HTML-like payloads and verify the browser renders them as text. Verify Secret materialized values never appear in bootstrap, HTML, JSON API, SSE, log, Artifact metadata, or browser storage.

- [ ] **Step 7: Add accessibility smoke**

Run axe on Dashboard, Workflow Start, Run Detail, Review Detail, and Attempt Log. Add keyboard tests for dialogs, tabs, focus return, and visible focus states.

- [ ] **Step 8: Add large-data smoke**

Exercise approximately 1,000 generated Jobs, multi-MiB JSON metadata/info boundaries, and tens-of-MiB Execution Log. Verify list/log virtualization prevents browser lockups.

- [ ] **Step 9: Run the full verification set**

```bash
pytest -q
cd webui
npm ci
npm test
npm run build
npx playwright test
```

Expected: all pass.

- [ ] **Step 10: Commit**

```bash
git add tests webui/e2e .github pyproject.toml
git commit -m "test: verify WebUI end to end"
```

---

### Task 16: Final Contract/Packaging Verification and Documentation Closeout

**Files:**
- Modify: `README.md`
- Modify: `docs/detailed-design/14-webui.md` only if implementation discovered a genuine approved-contract clarification; do not silently change behavior.
- Modify: release/build documentation once repository conventions exist.

**Interfaces:**
- Consumes: complete implementation.
- Produces: releasable, documented `jobrunner[web]` MVP.

- [ ] **Step 1: Verify forbidden capabilities do not exist**

Search source/UI/API for accidental additions:

```bash
grep -R "mark_success\|force_complete\|state_set\|state_delete\|runner.*restart\|runtime.*save\|workflow.*edit" jobrunner/adapters/web webui/src
```

Review every match. Expected: no public WebUI implementation of forbidden mutation; `runner restart` may appear only as read-only history text/data.

- [ ] **Step 2: Verify Web adapter isolation**

Ensure `jobrunner/adapters/web` does not import Repository implementations, SQLite modules, local Store path helpers, or directly open Run/Artifact/Log files.

- [ ] **Step 3: Verify OpenAPI/type drift**

Regenerate OpenAPI + TypeScript types and require a clean diff.

```bash
python -m <project_openapi_export_command>
cd webui
npm run api:generate
cd ..
git diff --exit-code
```

When implementing Task 7, replace `<project_openapi_export_command>` with the exact committed exporter module name and update this plan before execution continues; do not leave the placeholder in the committed implementation branch.

- [ ] **Step 4: Verify package boundaries**

Test both:

```bash
pip install .
pip install '.[web]'
```

Base install must not require FastAPI. Web install must serve the built SPA without Node.

- [ ] **Step 5: Update README usage**

Document Parent integration at the API level without inventing a standalone production server requirement. Include a minimal example using the actual final router factory signature from Task 5.

- [ ] **Step 6: Run full final suite**

```bash
pytest -q
cd webui && npm ci && npm test && npm run build && npx playwright test
```

- [ ] **Step 7: Commit**

```bash
git add README.md docs pyproject.toml jobrunner webui tests
git commit -m "docs: finalize WebUI MVP integration"
```

---

## Plan Self-Review Notes

### Spec coverage

The plan covers all `14-webui.md` implementation areas:

```text
FastAPI embedding/auth -> Tasks 5-6
Service/API additions -> Tasks 2-4
React architecture/routing -> Tasks 7-8
Workflow browse/start -> Task 9
Run/Job/Attempt/Step -> Task 10
Input/Output/State/Event -> Task 11
Review/Artifact/Runner/Runtime/Dashboard -> Task 12
SSE/polling/log -> Task 13
multi-mount/static/wheel -> Task 14
security/accessibility/E2E/performance -> Task 15
contract/package closeout -> Task 16
normative docs alignment -> Task 1
```

### Deliberate sequencing constraint

The current repository is design-only. This plan is **not the first executable implementation plan for JobRunner**; it is the implementation plan for the approved WebUI subsystem once the Core implementation described by `01`–`13` exists. Implementing the WebUI first would either fail immediately or force the Web adapter to become a duplicate Core, violating the design.

Therefore, if the Core code is still absent at execution time, the next action is to write and execute separate Core implementation plans before returning to this plan.
