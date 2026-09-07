# JobRunner Core Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the first executable JobRunner codebase foundation: package/test tooling, canonical JSON and strict JSON primitives, Actor/Authorization contracts, SQLite migration infrastructure with the canonical 18-table schema, strict Workflow Definition loading/validation, source-byte Resolver behavior, and Action/Validator registry metadata.

**Architecture:** This phase deliberately stops before Workflow scheduling, Runner processes, Attempt execution, Dynamic/Reusable execution, External/Human activation, Service/MCP/HTTP adapters, or WebUI. It establishes the immutable contracts those later phases depend on: strict typed models, canonical serialization/digests, durable schema, security identity, definition source handling, and registry identity/versioning.

**Tech Stack:** Python >=3.10; `ruamel.yaml >=0.19.1,<0.20`; `pydantic >=2.13,<3`; `jsonschema >=4.26,<5`; standard-library `json`, `hashlib`, `sqlite3`, `uuid`, `datetime`, `pathlib`; pytest for tests.

**Spec:** `docs/design.md`, `docs/detailed-design/01-workflow-definition.md`, `02-expression-and-inputs.md` (persistent input/secret reference primitives only), `08-persistence.md`, `12-security-and-secrets.md`, `13-testing.md`.

## Global Constraints

- Python >=3.10.
- Package root is `jobrunner/`; do not introduce a second `src/jobrunner` tree.
- Core package has no FastAPI/MCP dependency.
- Core models are strict/no-coercion. In particular: `"1" != 1`, `"true" != true`, `true` is not integer/number, `1.0` is not integer.
- Canonical JSON is exactly `jobrunner.canonical-json.v1`: `ensure_ascii=False`, `sort_keys=True`, separators `(',', ':')`, `allow_nan=False`, UTF-8, string object keys only, no Unicode normalization, no implicit conversion.
- Canonical timestamp persistence format is exactly `YYYY-MM-DDTHH:MM:SS.ffffffZ`.
- SQLite uses `foreign_keys=ON`, `busy_timeout=5000`, `journal_mode=WAL`.
- Canonical persistence table set is exactly 18 tables. This phase must not add convenience tables.
- Workflow authoring format is YAML 1.2 safe load. Reject duplicate keys, merge key `<<`, custom tags, unknown keys, YAML 1.1 boolean fallback, NaN/Infinity.
- JSON Schema is Draft 2020-12 only. `$ref`/`$dynamicRef` may only be same-document fragments beginning with `#`; external/relative/file/http references are rejected at Definition load.
- Every executable Workflow reference must resolve current UTF-8 YAML source bytes; typed-object-only executable definitions are not allowed.
- `wf_start` itself is not implemented in this phase, but Resolver APIs must make its required fresh-byte read possible.
- Authentication is parent responsibility; Core defines `ActorContext`, `AccessScope`, `AuthorizationProvider`, and default AllowAll only.
- `actor_principal_key` uses only `actor_type`, `actor_id`, and `access_scope`; `source`, `claims`, `metadata` are excluded.
- Secret values are not persisted or implemented as execution material in this phase. Only secret name/reference/binding data contracts are introduced.
- Use TDD. Every task ends with passing focused tests and a commit.

---

## Locked File Structure for This Phase

```text
pyproject.toml
jobrunner/
├─ __init__.py
├─ canonical_json.py
├─ errors.py
├─ ids.py
├─ time.py
├─ json_types.py
├─ registry.py
├─ definitions/
│  ├─ __init__.py
│  ├─ models.py
│  ├─ schema.py
│  ├─ loader.py
│  └─ resolver.py
├─ security/
│  ├─ __init__.py
│  ├─ models.py
│  ├─ providers.py
│  └─ principal.py
└─ persistence/
   ├─ __init__.py
   ├─ database.py
   ├─ migrations.py
   └─ sql/
      └─ 001_initial.sql

tests/
├─ test_canonical_json.py
├─ test_json_types.py
├─ test_time_and_ids.py
├─ definitions/
│  ├─ test_loader.py
│  ├─ test_schema.py
│  └─ test_resolver.py
├─ security/
│  └─ test_principal.py
├─ persistence/
│  └─ test_migrations.py
└─ test_registry.py
```

Later phases may add modules, but must not move these contracts without an explicit design change.

---

### Task 1: Bootstrap the Python Package and Test Tooling

**Files:**
- Create: `pyproject.toml`
- Create: `jobrunner/__init__.py`
- Create: `jobrunner/errors.py`
- Create: `tests/test_package.py`

**Interfaces:**
- Produces installable `jobrunner` package and common domain exception shape.

- [ ] **Step 1: Write the failing package import/version test**

Create `tests/test_package.py`:

```python
import jobrunner


def test_package_exposes_version() -> None:
    assert isinstance(jobrunner.__version__, str)
    assert jobrunner.__version__
```

- [ ] **Step 2: Create `pyproject.toml` with exact Core dependencies**

Use this project skeleton:

```toml
[build-system]
requires = ["setuptools>=75", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "jobrunner"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
  "ruamel.yaml>=0.19.1,<0.20",
  "pydantic>=2.13,<3",
  "jsonschema>=4.26,<5",
  "cel-python>=0.5,<0.6",
  "jmespath>=1.1,<2",
]

[project.optional-dependencies]
test = ["pytest>=8,<9", "pytest-cov>=5,<7"]
mcp = []
web = []
all = []

[tool.setuptools.packages.find]
include = ["jobrunner*"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra"
```

Do not add FastAPI or MCP runtime packages yet; later adapter plans populate those extras.

- [ ] **Step 3: Create package version and common exception**

`jobrunner/__init__.py`:

```python
__version__ = "0.1.0"
```

`jobrunner/errors.py`:

```python
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class JobRunnerError(Exception):
    code: str
    message: str
    retryable: bool = False
    field: str | None = None
    path: str | None = None
    details: Any = None

    def __str__(self) -> str:
        return f"{self.code}: {self.message}"
```

- [ ] **Step 4: Install and run the focused test**

```bash
python -m pip install -e '.[test]'
pytest tests/test_package.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml jobrunner/__init__.py jobrunner/errors.py tests/test_package.py
git commit -m "feat: bootstrap JobRunner package"
```

---

### Task 2: Implement Canonical JSON, Strict JSON Validation, IDs, and Time

**Files:**
- Create: `jobrunner/canonical_json.py`
- Create: `jobrunner/json_types.py`
- Create: `jobrunner/ids.py`
- Create: `jobrunner/time.py`
- Create: `tests/test_canonical_json.py`
- Create: `tests/test_json_types.py`
- Create: `tests/test_time_and_ids.py`

**Interfaces:**
- Produces:

```python
canonical_json_bytes(value: object) -> bytes
canonical_json_text(value: object) -> str
sha256_hex(data: bytes) -> str
canonical_json_digest(value: object) -> str
validate_json_value(value: object) -> None
validate_signed64(value: object, *, field: str) -> int
new_id(prefix: str) -> str
format_utc_timestamp(value: datetime) -> str
parse_utc_timestamp(value: str) -> datetime
```

- [ ] **Step 1: Write canonical JSON golden tests**

Create tests that assert exact bytes:

```python
from jobrunner.canonical_json import canonical_json_bytes, canonical_json_digest


def test_canonical_json_sorts_keys_and_preserves_unicode() -> None:
    value = {"z": 1.0, "a": "日本語", "n": 1}
    assert canonical_json_bytes(value) == b'{"a":"\xe6\x97\xa5\xe6\x9c\xac\xe8\xaa\x9e","n":1,"z":1.0}'


def test_canonical_json_distinguishes_integer_and_float() -> None:
    assert canonical_json_bytes(1) == b"1"
    assert canonical_json_bytes(1.0) == b"1.0"
    assert canonical_json_digest(1) != canonical_json_digest(1.0)
```

Add rejection tests for non-string object keys, tuple, set, bytes, Decimal, datetime, NaN, positive/negative Infinity.

- [ ] **Step 2: Write strict JSON-type tests**

Cover the canonical rules:

```python
import pytest

from jobrunner.json_types import is_json_integer, is_json_number, validate_json_value


def test_bool_is_not_integer_or_number() -> None:
    assert is_json_integer(True) is False
    assert is_json_number(True) is False


def test_float_is_not_integer() -> None:
    assert is_json_integer(1.0) is False


def test_tuple_is_not_json_array() -> None:
    with pytest.raises(TypeError):
        validate_json_value((1, 2))
```

- [ ] **Step 3: Implement strict recursive JSON validation before serialization**

`jobrunner/json_types.py` must explicitly branch on exact Python JSON-compatible types. Do not rely on `json.dumps(default=...)`.

Core implementation pattern:

```python
import math
from collections.abc import Mapping


def is_json_integer(value: object) -> bool:
    return type(value) is int


def is_json_number(value: object) -> bool:
    return type(value) is int or (type(value) is float and math.isfinite(value))


def validate_json_value(value: object) -> None:
    if value is None or type(value) in {bool, int, str}:
        return
    if type(value) is float:
        if not math.isfinite(value):
            raise TypeError("non-finite JSON number")
        return
    if type(value) is list:
        for item in value:
            validate_json_value(item)
        return
    if isinstance(value, Mapping) and type(value) is dict:
        for key, item in value.items():
            if type(key) is not str:
                raise TypeError("JSON object keys must be strings")
            validate_json_value(item)
        return
    raise TypeError(f"unsupported JSON value type: {type(value).__name__}")
```

- [ ] **Step 4: Implement canonical serialization/digest**

`jobrunner/canonical_json.py`:

```python
import hashlib
import json

from jobrunner.json_types import validate_json_value


def canonical_json_text(value: object) -> str:
    validate_json_value(value)
    return json.dumps(
        value,
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
        allow_nan=False,
    )


def canonical_json_bytes(value: object) -> bytes:
    return canonical_json_text(value).encode("utf-8")


def sha256_hex(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


def canonical_json_digest(value: object) -> str:
    return sha256_hex(canonical_json_bytes(value))
```

- [ ] **Step 5: Implement signed64, opaque ID, and canonical UTC timestamp helpers**

Use exact signed64 range `-(2**63)` through `2**63-1`. `new_id("wfr")` returns `wfr_` plus UUID4 lowercase hex. Timestamp formatter rejects naive datetimes and always emits UTC six-digit microseconds plus `Z`.

- [ ] **Step 6: Run focused tests**

```bash
pytest tests/test_canonical_json.py tests/test_json_types.py tests/test_time_and_ids.py -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/canonical_json.py jobrunner/json_types.py jobrunner/ids.py jobrunner/time.py tests/test_canonical_json.py tests/test_json_types.py tests/test_time_and_ids.py
git commit -m "feat: add canonical JSON and core primitives"
```

---

### Task 3: Implement Actor, AccessScope, Authorization, Secret Name/Binding, and Principal Identity

**Files:**
- Create: `jobrunner/security/__init__.py`
- Create: `jobrunner/security/models.py`
- Create: `jobrunner/security/providers.py`
- Create: `jobrunner/security/principal.py`
- Create: `tests/security/test_principal.py`
- Create: `tests/security/test_models.py`

**Interfaces:**
- Produces:

```python
ActorContext
AccessScope
SecretBinding
AuthorizationProvider.authorize(actor, operation, resource, scope) -> bool
AllowAllAuthorizationProvider
SecretsProvider.get(name, actor, scope) -> str
actor_principal_key(actor, scope) -> str
validate_secret_name(name: str) -> str
```

- [ ] **Step 1: Write principal-key golden tests**

Use fixed Actor/Scope values and calculate expected digest through the canonical JSON helper, not a second serialization implementation:

```python
from jobrunner.canonical_json import canonical_json_digest
from jobrunner.security.models import ActorContext, AccessScope
from jobrunner.security.principal import actor_principal_key


def test_actor_principal_key_uses_only_identity_and_scope() -> None:
    scope = AccessScope(value={"project": "p1"})
    actor_a = ActorContext(actor_type="user", actor_id="u1", source="web", claims={"role": "a"})
    actor_b = ActorContext(actor_type="user", actor_id="u1", source="mcp", claims={"role": "b"})
    expected = "apr_" + canonical_json_digest({
        "actor_type": "user",
        "actor_id": "u1",
        "access_scope": {"project": "p1"},
    })
    assert actor_principal_key(actor_a, scope) == expected
    assert actor_principal_key(actor_b, scope) == expected
```

Add tests showing different `actor_id` or AccessScope changes the principal key.

- [ ] **Step 2: Write strict model tests**

Reject empty `actor_type`, empty-string `actor_id`, non-JSON-compatible claims/metadata/scope, invalid Secret names, duplicate SecretBinding pointers when normalized by the helper introduced below.

- [ ] **Step 3: Implement strict Pydantic security models**

Use `ConfigDict(extra="forbid", strict=True, frozen=True)`. Represent AccessScope as a JSON-compatible object wrapper so it can be persisted and hashed without parent-specific schema assumptions.

```python
class AccessScope(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True, frozen=True)
    value: dict[str, Any]
```

Validate all nested JSON with `validate_json_value`.

- [ ] **Step 4: Implement provider contracts**

Use `Protocol` for provider interfaces and a concrete AllowAll default:

```python
class AuthorizationProvider(Protocol):
    def authorize(self, actor: ActorContext, operation: str, resource: object, scope: AccessScope) -> bool:
        raise NotImplementedError


class AllowAllAuthorizationProvider:
    def authorize(self, actor: ActorContext, operation: str, resource: object, scope: AccessScope) -> bool:
        return True
```

Define `SecretsProvider` contract but do not implement storage/backends in this phase.

- [ ] **Step 5: Implement Secret name and binding normalization**

Secret names match `^[A-Za-z_][A-Za-z0-9_]*$`. `normalize_secret_bindings()` sorts by RFC6901 pointer ascending and rejects duplicate pointers; values remain names/references only.

- [ ] **Step 6: Run tests**

```bash
pytest tests/security -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/security tests/security
git commit -m "feat: add security identity contracts"
```

---

### Task 4: Implement SQLite Connection and Migration Infrastructure with the Canonical 18 Tables

**Files:**
- Create: `jobrunner/persistence/__init__.py`
- Create: `jobrunner/persistence/database.py`
- Create: `jobrunner/persistence/migrations.py`
- Create: `jobrunner/persistence/sql/001_initial.sql`
- Create: `tests/persistence/test_migrations.py`

**Interfaces:**
- Produces:

```python
open_database(path: Path) -> sqlite3.Connection
apply_migrations(connection: sqlite3.Connection) -> None
current_schema_version(connection: sqlite3.Connection) -> int
```

- [ ] **Step 1: Write failing PRAGMA/table-set tests**

The test must create a temporary DB, call `open_database()` + `apply_migrations()`, and assert:

```python
EXPECTED_TABLES = {
    "schema_migrations",
    "workflow_runs",
    "job_runs",
    "job_attempts",
    "job_steps",
    "dynamic_expansions",
    "reusable_bindings",
    "workflow_state",
    "workflow_state_history",
    "artifacts",
    "events",
    "execution_logs",
    "runners",
    "runner_restarts",
    "external_tasks",
    "external_leases",
    "human_reviews",
    "idempotency_records",
}
```

Also assert `PRAGMA foreign_keys == 1`, `PRAGMA busy_timeout == 5000`, and journal mode is `wal` for file-backed databases.

- [ ] **Step 2: Write idempotent migration tests**

Calling `apply_migrations()` twice must leave one `schema_migrations` row for version 1 and the exact same 18-table set.

- [ ] **Step 3: Implement `open_database()`**

Use `sqlite3.connect(..., isolation_level=None)` or an equivalent explicit-transaction strategy. Apply PRAGMAs immediately on each connection. Configure rows consistently (`sqlite3.Row`) for later Repository phases.

- [ ] **Step 4: Create `001_initial.sql` from detailed design 08**

The migration must define all 18 tables and every SQL CHECK/index explicitly specified in `08`. Do not invent a Workflow Definition history table, Concurrency table, notification table, or Dynamic template `job_runs` row/table.

At minimum, verify these named indexes/constraints in tests because later phases depend on them:

```text
one running Step per Attempt partial unique index
external available-task index
one active external Lease per Task partial unique index
review status/created ordering index
idempotency expiry index
runner restart query indexes from design 08
job scheduling indexes from design 08
```

- [ ] **Step 5: Implement migration runner**

`apply_migrations()` runs each unapplied migration inside an explicit transaction and inserts `schema_migrations(version,name,applied_at)` only after the migration statements succeed. Any statement failure rolls back the whole version.

- [ ] **Step 6: Add schema invariant spot-check tests**

Test representative DB constraints from design 08, including:

```text
workflow version >= 1
boolean columns only 0/1 where CHECK is specified
runner pid > 0 where specified
review outcome/status CHECKs where specified
one active lease per task
one running step per attempt
```

Runtime-only invariants that design 08 explicitly leaves to Repository/Service are not duplicated as new SQL CHECKs here.

- [ ] **Step 7: Run persistence tests**

```bash
pytest tests/persistence/test_migrations.py -q
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add jobrunner/persistence tests/persistence/test_migrations.py
git commit -m "feat: add canonical SQLite schema"
```

---

### Task 5: Implement Strict Workflow Definition Models and YAML 1.2 Loader

**Files:**
- Create: `jobrunner/definitions/__init__.py`
- Create: `jobrunner/definitions/models.py`
- Create: `jobrunner/definitions/loader.py`
- Create: `tests/definitions/test_loader.py`

**Interfaces:**
- Produces:

```python
WorkflowDefinition
WorkflowInputDefinition
WorkflowConcurrencyDefinition
WorkflowSettings
JobDefinition
load_workflow_yaml(source: bytes) -> WorkflowDefinition
workflow_definition_tree(definition: WorkflowDefinition) -> dict[str, object]
workflow_definition_hash(definition: WorkflowDefinition) -> str
```

This task models the full field shape in detailed design 01, but expression evaluation and execution behavior remain later-phase concerns.

- [ ] **Step 1: Write loader rejection tests first**

Fixtures must cover:

```text
duplicate key
merge key <<
custom YAML tag
unknown top-level key
unknown Job key
YAML 1.1 boolean-like scalar such as on/off/yes/no remaining strings under YAML 1.2
version as string
version as bool
priority as string/bool
NaN/Infinity
missing name/version/jobs
empty workflow name
```

- [ ] **Step 2: Write strict Input/default tests**

Test that a Workflow Input definition rejects default `"1"` for integer, `1` for string, `true` for integer/number, and null unless `nullable=true`.

- [ ] **Step 3: Implement safe YAML 1.2 decoding with duplicate/merge/custom-tag rejection**

Configure `ruamel.yaml` in safe YAML 1.2 mode. Convert parser failures into `JobRunnerError(code="workflow_definition_invalid", ...)` or the more specific canonical code defined by the Core error taxonomy when already available.

Reject merge key before typed-model construction even when the loader could otherwise expand it.

- [ ] **Step 4: Implement immutable strict Pydantic models**

Every definition model uses:

```python
model_config = ConfigDict(extra="forbid", strict=True, frozen=True)
```

Use explicit validators for signed64 ranges, non-empty strings, exact enum values, executor field exclusions, Human output-schema prohibition, Reusable field exclusions, Concurrency group/max/on-limit rules, and settings ranges defined in design 01.

- [ ] **Step 5: Implement deterministic runtime-independent definition tree/hash**

`workflow_definition_tree()` converts the typed model to a JSON-compatible tree without source path/runtime/cache metadata. `workflow_definition_hash()` returns lowercase SHA-256 hex of `canonical_json_bytes(tree)`.

- [ ] **Step 6: Add hash stability tests**

Equivalent YAML mappings with different key order must produce the same Definition hash. `version: 1` versus any invalid string form must not be normalized into the same definition.

- [ ] **Step 7: Run tests**

```bash
pytest tests/definitions/test_loader.py -q
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add jobrunner/definitions tests/definitions/test_loader.py
git commit -m "feat: add strict workflow definition loader"
```

---

### Task 6: Implement Draft 2020-12 Schema Validation and Strict Workflow Input Validation

**Files:**
- Create: `jobrunner/definitions/schema.py`
- Modify: `jobrunner/definitions/models.py`
- Modify: `jobrunner/definitions/loader.py`
- Create: `tests/definitions/test_schema.py`

**Interfaces:**
- Produces:

```python
validate_schema_definition(schema: dict[str, object]) -> None
validate_workflow_inputs(definition: WorkflowDefinition, values: dict[str, object]) -> dict[str, object]
```

- [ ] **Step 1: Write schema-draft/ref tests**

Accept:

```text
$schema omitted
$schema exactly https://json-schema.org/draft/2020-12/schema
$defs + $ref beginning with #
$dynamicRef beginning with #
```

Reject with `json_schema_external_ref_forbidden`:

```text
https://example.com/schema.json
file:///tmp/schema.json
../schema.json
schema.json#/x
```

Reject older/newer `$schema` URIs.

- [ ] **Step 2: Implement recursive ref scan**

Before constructing/using `Draft202012Validator`, recursively inspect every object/list node. For keys `$ref` and `$dynamicRef`, require a string beginning with `#`.

- [ ] **Step 3: Implement Draft 2020-12 schema check**

Call `Draft202012Validator.check_schema(schema)` after the external-ref scan. Do not configure a registry/retriever that can fetch network or filesystem resources.

- [ ] **Step 4: Write Workflow Input validation-order tests**

Test exact order/semantics:

```text
extra key -> reject
required key missing -> reject
null when nullable=false -> reject
null when nullable=true -> accept without non-null schema validation
wrong strict base type -> reject before JSON Schema
valid strict base type but schema failure -> reject
optional missing without default -> remains absent
optional missing with default -> exact typed default inserted
```

- [ ] **Step 5: Implement strict Workflow Input validation**

Return a new plain dict snapshot. Do not mutate caller input. Apply definition defaults only; no adapter/UI defaults.

- [ ] **Step 6: Run tests**

```bash
pytest tests/definitions/test_schema.py tests/definitions/test_loader.py -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/definitions tests/definitions/test_schema.py
git commit -m "feat: add workflow schema and input validation"
```

---

### Task 7: Implement Workflow Resolver with Fresh UTF-8 Source Bytes

**Files:**
- Create: `jobrunner/definitions/resolver.py`
- Create: `tests/definitions/test_resolver.py`

**Interfaces:**
- Produces:

```python
@dataclass(frozen=True)
class ResolvedWorkflowSource:
    workflow_id: str
    source_bytes: bytes
    source_kind: str
    source_display: str
    base_directory: Path | None

class WorkflowResolver(Protocol):
    def list_refs(self) -> list[str]:
        raise NotImplementedError

    def resolve_source(self, workflow_ref: str) -> ResolvedWorkflowSource:
        raise NotImplementedError
```

Provide at least a filesystem resolver and an in-memory/registered-source resolver for tests/embedding. Registered executable refs must store source bytes, not only typed definitions.

- [ ] **Step 1: Write fresh-read tests**

Create a temporary YAML file, resolve it, replace contents without relying on a changed mtime/size, resolve again, and assert the second call returns the new bytes. The resolver may cache browse metadata, but `resolve_source()` must read authoritative current bytes.

- [ ] **Step 2: Write invalid encoding/path tests**

Reject non-UTF-8 source, missing source, directory path, and filesystem path escape outside configured root. Resolve symlinks before root containment checks.

- [ ] **Step 3: Implement filesystem source resolution**

Return canonical workflow ID plus exact bytes and the resolved base directory. Do not parse YAML inside the Resolver; loader/parsing remains a separate responsibility.

- [ ] **Step 4: Implement registered source resolution**

Registration requires a callable/source provider that returns UTF-8 YAML bytes for each read. A prebuilt `WorkflowDefinition` alone is rejected as an executable source registration.

- [ ] **Step 5: Add browse-cache non-authority test**

If a list/info cache contains an older valid Definition but the current source becomes invalid/unavailable, an execution-path `resolve_source()` + `load_workflow_yaml()` must fail and must not return the cached Definition.

- [ ] **Step 6: Run tests**

```bash
pytest tests/definitions/test_resolver.py -q
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add jobrunner/definitions/resolver.py tests/definitions/test_resolver.py
git commit -m "feat: add workflow source resolver"
```

---

### Task 8: Implement Action/Validator Registry Identity and Version Contracts

**Files:**
- Create: `jobrunner/registry.py`
- Create: `tests/test_registry.py`

**Interfaces:**
- Produces:

```python
ActionRegistration
ValidatorRegistration
Registry.register_action(action_id, version, callable, uses_runtime=False, metadata=None) -> None
Registry.register_validator(validator_id, version, callable) -> None
Registry.get_action(action_id) -> ActionRegistration
Registry.get_validator(validator_id) -> ValidatorRegistration
Registry.snapshot_action_versions(ids: set[str]) -> dict[str, str]
Registry.snapshot_validator_versions(ids: set[str]) -> dict[str, str]
```

- [ ] **Step 1: Write duplicate/current-version tests**

Within one Registry process instance:

```text
same Action ID registered twice -> reject
same Validator ID registered twice -> reject
one current version only
empty ID/version -> reject
uses_runtime must be exact bool
metadata must be JSON-compatible when present
```

- [ ] **Step 2: Write callable-arity tests**

At registration/bootstrap time:

```text
uses_runtime=false -> callable must accept one positional execution_input
uses_runtime=true -> callable must accept execution_input + runtime_handle
varargs allowed but uses_runtime remains source of truth
```

Do not infer `uses_runtime` from callable signature.

- [ ] **Step 3: Implement immutable registration records and Registry**

Use exact caller-supplied version string/identity without normalization. Store callable only process-locally; later persistence stores ID/version snapshot, never pickled callable.

- [ ] **Step 4: Implement version snapshot helpers**

Snapshot maps must be deterministic plain dicts suitable for canonical JSON persistence. Missing requested registrations fail closed.

- [ ] **Step 5: Run tests**

```bash
pytest tests/test_registry.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add jobrunner/registry.py tests/test_registry.py
git commit -m "feat: add action and validator registry"
```

---

### Task 9: Foundation Acceptance and Cross-Contract Regression Suite

**Files:**
- Create: `tests/test_foundation_acceptance.py`
- Modify: `README.md`

**Interfaces:**
- Consumes all phase-one modules.
- Produces a stable foundation gate for the next Runtime/Scheduling plan.

- [ ] **Step 1: Write an end-to-end foundation acceptance test**

The test must:

1. create a temp SQLite DB and apply migration 1;
2. register a one-argument Action and a Validator;
3. write a valid Workflow YAML file using strict inputs, local `$defs/$ref`, priority/concurrency/settings, and one internal Job;
4. resolve current source bytes;
5. load the strict `WorkflowDefinition`;
6. validate Workflow Inputs;
7. compute Definition hash and canonical input digest source;
8. build `ActorContext`/`AccessScope` and compute `actor_principal_key`;
9. verify Registry version snapshots are JSON-compatible;
10. assert the DB still contains exactly the canonical 18 tables.

No Workflow Run row or Job execution is created in this phase.

- [ ] **Step 2: Add negative integration cases**

In the same test module verify:

```text
source becomes invalid after browse -> fresh load fails
external JSON Schema ref -> fail closed
bool supplied to integer Workflow input -> reject
Actor source/claims change -> principal key unchanged
Actor ID/scope change -> principal key changes
second application of migration 1 -> no schema drift
```

- [ ] **Step 3: Run the complete phase suite**

```bash
pytest -q
```

Expected: all phase-one tests PASS.

- [ ] **Step 4: Verify package build/install**

```bash
python -m pip install build
python -m build
python -m pip install --force-reinstall dist/jobrunner-0.1.0-py3-none-any.whl
python -c "import jobrunner; print(jobrunner.__version__)"
```

Expected output includes `0.1.0`.

- [ ] **Step 5: Update README project status**

Document that Foundation is implemented while Runtime/Scheduling, Runner/IPC, executors, Service adapters, and WebUI remain subsequent implementation phases. Do not claim Workflow execution works yet.

- [ ] **Step 6: Commit**

```bash
git add tests/test_foundation_acceptance.py README.md
git commit -m "test: lock JobRunner foundation contracts"
```

---

## Self-Review Checklist

### Spec coverage for this phase

- Package/dependency boundaries: Task 1
- canonical-json-v1 / strict JSON / signed64 / timestamps / IDs: Task 2
- Actor/AccessScope/Authorization/Secret binding primitives: Task 3
- canonical 18-table persistence + migration behavior: Task 4
- YAML 1.2 strict Workflow Definition: Task 5
- Draft 2020-12 self-contained Schema + Workflow Input validation: Task 6
- executable source bytes / fresh Resolver reads / fail-closed cache behavior: Task 7
- Action/Validator identity/version/uses_runtime metadata: Task 8
- cross-contract acceptance: Task 9

### Explicitly deferred to later implementation plans

This plan does **not** implement:

```text
CEL/JMESPath evaluation engine
persistent Job Input activation
Workflow Run creation/admission/scheduling
Concurrency transactions
Runner Pool/Runner/Action Runner/IPC
Attempt/Step execution
Secret materialization/SecretGuard streaming
PayloadStore/ArtifactStore runtime operations
Dynamic Jobs
Reusable Workflows
External Tasks/Human Reviews
Retry/Recovery/Cancel runtime behavior
State/Event/Log runtime behavior
Service API/MCP/HTTP
FastAPI/React WebUI
```

Those depend on the contracts established here and should be planned as separate executable phases rather than folded into one oversized implementation plan.
