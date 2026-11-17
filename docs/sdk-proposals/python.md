# Yily Python SDK — Architecture Proposal

## 1. Objective

Introduce a lightweight, fully typed Python SDK that lets applications retrieve configuration variables from Yily and use them directly in Python code: scripts, web services (FastAPI, Django, Flask), workers and CI jobs.

The SDK follows Yily's philosophy — **simple and effective** — and uses the existing SDKs as its behavioral references:

- [`yily-sdk-go`](https://github.com/yannick2009/yily-sdk-go)
- [`yily-sdk-js`](https://github.com/yannick2009/yily-sdk-js)

The proposal is split in two parts:

- **The core** (sections 2 to 10) reproduces the behavior of the Go and JS/TS SDKs exactly, with Python naming.
- **Python-specific additions** (section 11) are optional ideas, presented separately so they can be discussed, adjusted or dropped without affecting the core.

## 2. Scope

The SDK is **read-only**. It supports:

- Connecting to a Yily instance and selecting an environment.
- Authenticating with a pre-provisioned, read-scoped Personal Access Token (PAT).
- Retrieving one variable's active value (`get`).
- Retrieving one variable at a specific version (`get_version`).
- Retrieving every active variable of the environment (`get_all`).
- Loading variables into a typed structure (`load`).
- Batch retrieval through the existing values endpoint.

Out of scope: creating, updating or deleting variables, and any project, environment, tag or token management. PATs are created and managed in the Yily web interface.

## 3. Alignment with the Existing SDKs

| Behavior | Go | JS/TS | Python (proposed) |
|---|---|---|---|
| Active value | `Get(ctx, name)` | `get(name)` | `get(name)` |
| Specific version | `GetVersion(ctx, name, v)` | `getVersion(name, v)` | `get_version(name, version)` |
| All active values | `GetAll(ctx)` | `getAll()` | `get_all()` |
| Typed loading | `Load(ctx, &struct)` + `yily` tags | `load(schema)` | `load(DataclassType)` + `yily.Var` |
| Default timeout | 5 s | 5000 ms | 5.0 s |
| Custom transport | `HTTPClient` | `fetch` | `http_client` (`httpx.Client`) |
| Cancellation | `context.Context` | `AbortSignal` | per-call `timeout`; task cancellation in async (section 11) |
| Errors | sentinel errors + `*APIError` | `YilyError` with `code` + `APIError` | `YilyError` with `code` + `APIError` |

Every rule described in sections 6 to 9 (validation, name normalization, deduplication, missing variables, conversion grammar, error codes) mirrors the Go and JS implementations, so the three SDKs behave the same against the same server.

## 4. Project Structure

Like the other SDKs, the Python SDK lives in a dedicated repository: `github.com/yannick2009/yily-sdk-python`.

```text
.
├── pyproject.toml
├── README.md
├── CHANGELOG.md
├── LICENSE
├── src/
│   └── yily/
│       ├── __init__.py      # Public exports
│       ├── py.typed         # PEP 561 marker: the package ships type hints
│       ├── _config.py       # Config validation and base URL normalization
│       ├── _client.py       # Client: get, get_version, get_all, load
│       ├── _http.py         # Request building, envelope parsing, error mapping
│       ├── _loader.py       # Dataclass inspection and field mapping
│       ├── _convert.py      # String → Python type conversion, duration parser
│       ├── _utils.py        # Variable name normalization
│       └── errors.py        # YilyError, APIError, YilyErrorCode
├── tests/
└── examples/
```

Modules prefixed with `_` are private. The public surface is what `yily/__init__.py` re-exports: `Client`, `Var`, `YilyError`, `APIError` and `YilyErrorCode`.

## 5. Public API

```python
class Client:
    def __init__(
        self,
        *,
        base_url: str,
        environment_id: str,
        token: str,
        timeout: float = 5.0,
        http_client: httpx.Client | None = None,
    ) -> None: ...

    def get(self, name: str) -> str: ...
    def get_version(self, name: str, version: int) -> str: ...
    def get_all(self) -> dict[str, str]: ...
    def load(self, target: type[T]) -> T: ...

    def close(self) -> None: ...
    def __enter__(self) -> Self: ...
    def __exit__(self, *exc_info: object) -> None: ...
```

Minimal example:

```python
import os
import yily

with yily.Client(
    base_url="https://yily.example.com",
    environment_id="ENVxxxxxxxxxxxx",
    token=os.environ["YILY_TOKEN"],
) as client:
    api_key = client.get("API_KEY")
    legacy_key = client.get_version("API_KEY", 1)
    all_vars = client.get_all()
```

A `Client` is immutable after construction and safe to share across threads, as in Go and JS. The context manager only closes the underlying HTTP connection pool. When `http_client` is provided, the SDK never closes it: its owner does.

## 6. Configuration and Authentication

### 6.1. Validation

The constructor validates its arguments eagerly and raises `YilyError` with the same codes as the JS SDK:

| Rule | Error code |
|---|---|
| `base_url` empty | `MISSING_BASE_URL` |
| `base_url` not an absolute `http`/`https` URL, or containing credentials, a query string or a fragment | `INVALID_BASE_URL` |
| `token` empty | `MISSING_TOKEN` |
| `token` not starting with `yily_`, or equal to `yily_` | `INVALID_TOKEN` |
| `environment_id` empty | `MISSING_ENVIRONMENT_ID` |
| `timeout` not a finite number | `INVALID_TIMEOUT` |

Surrounding whitespace is trimmed from every value. A zero or negative `timeout` selects the 5-second default, as in the other SDKs. The environment ID is treated as an opaque string and URL-encoded in request paths.

### 6.2. Base URL normalization

Trailing slashes are removed, one trailing `/api/v1` suffix is dropped, and exactly one `/api/v1` is appended. `https://yily.example.com`, `https://yily.example.com/` and `https://yily.example.com/api/v1` are therefore equivalent.

### 6.3. Authentication

Every request carries:

```http
Authorization: Bearer yily_<token>
Accept: application/json
```

A read-scoped PAT is recommended for applications. The token is never logged, never included in error messages and excluded from `repr(client)`.

## 7. Retrieving Variables

### 7.1. Batch endpoint

`get`, `get_version` and `load` use the read-only batch endpoint, like the Go and JS SDKs:

```http
POST /api/v1/environments/{environmentID}/variables/values
Content-Type: application/json

{
  "variables": [
    { "name": "DB_HOST", "version": 2 },
    { "name": "DB_PORT" }
  ]
}
```

```json
{
  "status": 200,
  "message": "Variable values retrieved successfully",
  "data": { "DB_HOST": "localhost", "DB_PORT": "5432" }
}
```

`get` and `get_version` send a batch containing one variable. `load` sends one batch for every mapped field.

### 7.2. Selection rules

Before a request is sent, the SDK applies the same rules as the Go and JS SDKs:

1. **Name normalization**: names are upper-cased and each run of whitespace becomes one underscore (`"db host"` → `DB_HOST`), matching the server's convention.
2. **Empty names**: `get` and `get_version` raise `INVALID_VARIABLE_NAME` when the normalized name is empty.
3. **Versions**: `get_version` raises `INVALID_VERSION` when `version` is not a positive integer. An omitted version selects the active version.
4. **Deduplication**: identical selections are sent once. The same variable requested at two different versions raises `CONFLICTING_VARIABLE_VERSION`.

### 7.3. Results

- A requested name absent from `data` raises `VARIABLE_NOT_FOUND`.
- A variable without an active version is returned as an empty string, which is a valid value.
- A malformed response (non-object envelope, non-string values) raises `DECODE_FAILED`.
- Keys are copied into a plain `dict`, so server-provided names cannot affect anything other than the returned mapping.

### 7.4. `get_all`

`get_all()` performs two requests, like the Go and JS SDKs:

1. `GET /api/v1/environments/{environmentID}/variables` to list variable names.
2. One batch request for their active values.

An empty environment returns `{}` after the first request, without a second one. Because there are two requests, the result is not an atomic snapshot: a variable created or deleted in between may be missing or absent.

## 8. Typed Loading

### 8.1. Declaring the structure

The Go SDK maps struct fields with `yily` tags; the JS SDK uses a schema object. The idiomatic Python equivalent is a **dataclass** whose fields are mapped with `typing.Annotated` and `yily.Var`:

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Annotated

import yily


@dataclass(frozen=True)
class AppConfig:
    db_host: Annotated[str, yily.Var("DB_HOST")]
    db_port: Annotated[int, yily.Var("DB_PORT")]
    debug: Annotated[bool, yily.Var("DEBUG")]
    timeout: Annotated[timedelta, yily.Var("TIMEOUT")]
    legacy_key: Annotated[str, yily.Var("API_KEY", version=2)]


config = client.load(AppConfig)
```

Mapping rules, aligned with the Go tags and JS schema:

- Only fields annotated with `yily.Var` are loaded. Other fields are ignored, as untagged Go fields are. They must have a default value, since `load` builds the instance.
- `yily.Var(name, version=N)` selects a specific version; `version` must be a positive integer, otherwise `INVALID_FIELD` is raised.
- Two fields mapping the same variable with different versions raise `CONFLICTING_VARIABLE_VERSION`.
- `load` accepts a dataclass **type** and returns a new instance, as the JS SDK returns a new object. Passing anything else raises `INVALID_FIELD`.

### 8.2. Loading pipeline

1. Inspect the dataclass and collect the mapped fields.
2. Retrieve every value in one batch request.
3. Check that every variable is present; otherwise raise `VARIABLE_NOT_FOUND` naming the variable.
4. Convert every value.
5. Build the instance only if every step succeeded. A failure never returns a partially loaded object.

### 8.3. Supported types and conversion grammar

The SDK follows the JS SDK's conversion grammar, which reproduces Go's `strconv` and `time.ParseDuration` rules (Go additionally accepts hexadecimal floats, which neither the JS nor the Python SDK supports). Python's built-in `int()` and `float()` are more permissive (they accept surrounding whitespace and `1_000`), so the SDK validates the input against the same patterns as the JS SDK before converting it.

| Go type | JS type | Python annotation | Accepted values |
|---|---|---|---|
| `string` | `string` | `str` | any value, including empty |
| `int`, `int64` | `int`, `int64` | `int` | optional sign followed by digits (`^[+-]?\d+$`), within the signed 64-bit range |
| `float64` | `float64` | `float` | decimal and exponent notation, plus `nan`, `inf`, `infinity` with an optional sign (case-insensitive) |
| `bool` | `bool` | `bool` | `1`, `t`, `T`, `TRUE`, `true`, `True` / `0`, `f`, `F`, `FALSE`, `false`, `False` |
| `time.Duration` | `duration` | `datetime.timedelta` | Go duration syntax: `0`, `5s`, `250ms`, `1h30m`, `1.5h`, units `ns`, `us`, `µs`, `ms`, `s`, `m`, `h` |

Python has a single integer type, so Go's `int` and `int64` both map to `int`. The 64-bit range check keeps behavior identical to Go and JS.

An invalid value raises `INVALID_VALUE`; an annotation outside this table raises `UNSUPPORTED_TYPE`. Messages name the variable and the expected type, **never the value**.

## 9. Error Handling

### 9.1. Error classes

The Python SDK follows the JS SDK model: one base exception carrying a stable code, and a subclass for API responses.

```python
class YilyError(Exception):
    code: YilyErrorCode          # stable, machine-readable SDK code

class APIError(YilyError):
    status_code: int             # HTTP status of the response
    api_code: str | None         # server's `code` field, when present
```

`YilyErrorCode` is a `StrEnum` whose values are **identical** to the JS SDK codes:

| Group | Codes |
|---|---|
| Configuration | `MISSING_BASE_URL`, `INVALID_BASE_URL`, `MISSING_TOKEN`, `INVALID_TOKEN`, `MISSING_ENVIRONMENT_ID`, `INVALID_TIMEOUT` |
| Loading and selection | `INVALID_FIELD`, `INVALID_VARIABLE_NAME`, `INVALID_VERSION`, `CONFLICTING_VARIABLE_VERSION`, `UNSUPPORTED_TYPE`, `INVALID_VALUE`, `VARIABLE_NOT_FOUND` |
| Transport and API | `REQUEST_FAILED`, `ENCODE_FAILED`, `DECODE_FAILED`, `API_ERROR`, `UNAUTHORIZED`, `NOT_FOUND` |

The JS-only `INVALID_FETCH` code has no Python equivalent, since `http_client` is type-checked.

### 9.2. Mapping API responses

As in the Go and JS SDKs:

- Any non-2xx response raises `APIError` with `status_code` and, when the body is a Yily envelope, the server's `code` as `api_code` (for example `VARIABLE_VERSION_NOT_FOUND` or `ENVIRONMENT_NOT_FOUND`).
- `code` is `UNAUTHORIZED` for HTTP 401, `NOT_FOUND` for HTTP 404, and `API_ERROR` otherwise.
- When the body is not a valid envelope, the error falls back to the HTTP status and reason phrase.
- Network failures and timeouts raise `YilyError` with `REQUEST_FAILED`, chaining the original `httpx` exception through `__cause__`.

```python
try:
    api_key = client.get("API_KEY")
except yily.APIError as error:
    log.error("Yily API error %s (%s)", error.status_code, error.api_code)
except yily.YilyError as error:
    if error.code is yily.YilyErrorCode.VARIABLE_NOT_FOUND:
        ...
    raise
```

## 10. HTTP and Security

- **Transport**: [`httpx`](https://www.python-httpx.org/), the only runtime dependency, with a 5-second default timeout and connection pooling.
- **No automatic retries**: neither the Go nor the JS SDK retries, so the core does not either.
- **HTTPS**: required outside local development, so PATs and values are protected in transit.
- **Redirects are not followed**, which is the `httpx` default. A Yily API never needs to redirect a client, and following a redirect after a misconfigured reverse proxy could send the request to another host. A `3xx` response raises an `APIError` that names the `Location` target, so the user can fix `base_url`.
- **No logging of secrets**: the SDK returns values to the caller and never logs, caches or persists them. It does not log anything by default.
- **Token revocation**: a password recovery on the server revokes every PAT. Applications then receive `UNAUTHORIZED` and need a new token.

## 11. Python-specific Additions (for discussion)

The following ideas do not exist in the Go or JS SDKs. Each one is independent, builds on the core without changing its behavior, and can be accepted, postponed or dropped.

| Addition | Why it fits Python | Effect on the core |
|---|---|---|
| `AsyncClient` | FastAPI, Starlette and async workers are common. `httpx` provides sync and async transports behind one API. | Same methods as coroutines. Cancellation works through task cancellation, the Python counterpart of `AbortSignal`. |
| Per-call `timeout=` | Python has no `context.Context`; a per-call override replaces it for long `get_all` calls. | Optional keyword argument, default unchanged. |
| `Client.from_env()` | Python services are usually configured through environment variables. | Explicit opt-in constructor reading `YILY_URL`, `YILY_TOKEN` and `YILY_ENVIRONMENT_ID`; nothing is read implicitly. |
| Pydantic support in `load` | Pydantic is the most common configuration library in Python. | Optional extra (`pip install "yily-sdk[pydantic]"`): values are fetched as for dataclasses, then validated by the model. |
| `get_many(names)` | Avoids N requests when a few values are needed without a dataclass. | One batch request, same selection rules as `load`. |
| Optional fields (`X \| None`, defaults) | Common pattern in Python configuration. | Would differ from Go and JS, where a missing variable is always an error. Proposed only if maintainers want it for all SDKs. |

## 12. Testing

The test suite mirrors the existing Go and JS tests so the three SDKs are checked against the same cases:

- Configuration validation and every configuration error code.
- Base URL and variable name normalization.
- `get`, `get_version`, `get_all`, including the empty-environment short-circuit.
- Batch deduplication and version conflicts.
- Missing variables and empty values.
- Every conversion type, including the boundary cases of the JS tests (64-bit limits, `nan`/`inf`, Go duration grammar).
- API errors with and without an envelope, 401/404 mapping, network failures and timeouts.
- `load` never returning a partially loaded object.

Tools: `pytest`, with [`respx`](https://lundberg.github.io/respx/) to mock `httpx` requests. Static checks use `mypy --strict` and `ruff`. The CI matrix covers every supported Python version on Linux, with one macOS and one Windows job. An integration job runs the examples against the official Yily Docker image (`deploy/docker`).

## 13. Packaging and Dependencies

- **Python**: 3.11+ (needed for `enum.StrEnum` and `typing.Self`; 3.10 can be supported with small compatibility shims if maintainers prefer).
- **Runtime dependency**: `httpx` only. Test and development tools are not installed by default.
- **Distribution**: published on PyPI as `yily-sdk` (the import name is `yily`), following the `yily-sdk-js` naming.
- **Build**: `pyproject.toml` with `hatchling`, `src/` layout, `py.typed` marker; development with `uv`.
- **Release**: semantic versioning, changelog, and PyPI publication from GitHub Actions using Trusted Publishing, so no long-lived PyPI token is stored.
- **Server compatibility**: requires a Yily server providing the batch values endpoint with per-variable versions, like the other SDKs.

## 14. Usage Examples

### 14.1. Script or CI job

```python
import os
import yily

with yily.Client(
    base_url=os.environ["YILY_URL"],
    environment_id=os.environ["YILY_ENVIRONMENT_ID"],
    token=os.environ["YILY_TOKEN"],
) as client:
    database_url = client.get("DATABASE_URL")
```

### 14.2. Typed configuration at startup

```python
client = yily.Client(base_url=..., environment_id=..., token=...)

try:
    config = client.load(AppConfig)
except yily.YilyError as error:
    raise SystemExit(f"Cannot load configuration from Yily: {error}") from error
finally:
    client.close()
```

### 14.3. Django `settings.py`

```python
import os
import yily

with yily.Client(
    base_url=os.environ["YILY_URL"],
    environment_id=os.environ["YILY_ENVIRONMENT_ID"],
    token=os.environ["YILY_TOKEN"],
) as _yily:
    _values = _yily.get_all()

SECRET_KEY = _values["SECRET_KEY"]
DATABASE_URL = _values["DATABASE_URL"]
```

## 15. Documentation Plan

The repository README follows the structure of the `yily-sdk-js` README so users switching languages find the same sections: requirements, installation, quickstart, configuration table, API reference, typed loading, error codes, security notes.

A documentation site can be added once the API is stable (MkDocs with `mkdocstrings`, which builds the API reference from docstrings and type hints). The README examples are executed in CI so they stay correct.

| Page | Content |
|---|---|
| Getting started | Installation, creating a **read-scoped** PAT in the web interface, finding the environment ID, first `get()` call. |
| Configuration | Every constructor option, validation rules, base URL normalization, timeouts, custom `httpx` client. |
| Retrieving variables | `get`, `get_version`, `get_all`, name normalization, missing and empty values. |
| Typed loading | Dataclasses with `yily.Var`, supported types and accepted formats, error reporting. |
| Error handling | `YilyError`, `APIError`, the full code table, examples. |
| Framework guides | Django, Flask, FastAPI and Celery, plus async usage if `AsyncClient` is accepted. |
| CI/CD | Storing the PAT as a CI secret, reading values in GitHub Actions and GitLab CI jobs. |
| Security best practices | Read-scoped tokens, expiry and rotation, HTTPS, never logging values, behavior after a password recovery. |
| Migrating from `.env` files | Moving existing variables with Yily's `.env` import, then replacing `python-dotenv` calls. |

## 16. Roadmap

The core comes first and must match the Go and JS SDKs before anything else is added.

| Phase | Content | Release |
|---|---|---|
| **1. Foundation** | Repository setup (`pyproject.toml`, `ruff`, `mypy --strict`, CI), configuration validation, HTTP layer, envelope parsing, error classes and codes. | — |
| **2. Core parity** | `get`, `get_version`, `get_all`, `load` for dataclasses with the full conversion grammar. Test suite mirroring the Go and JS tests. README. | `0.1.0` |
| **3. Python additions** | Only the additions from section 11 approved by the maintainers (for example `AsyncClient`, `from_env`, Pydantic). | `0.2.0` |
| **4. Stable release** | Integration tests against the Docker image, documentation site, PyPI Trusted Publishing, API freeze. | `1.0.0` |

## 17. Open Questions

1. **Batches above 1,000 variables.** The server accepts at most 1,000 selections per request, and neither existing SDK splits larger batches, so `get_all` on a very large environment gets a 400 error. Should all SDKs split batches, or is this limit acceptable?
2. **Minimum Python version**: 3.11, or 3.10 with compatibility shims?
3. **Package name**: `yily-sdk` on PyPI, or `yily-sdk-python` to match the repository name?
4. **Which Python-specific additions** from section 11 should be part of the first releases?

## 18. Design Principles

- **Parity first**: same behavior, validation rules and error codes as the Go and JS SDKs.
- **Read-only**: configuration retrieval only.
- **Simplicity**: four methods covering every use case.
- **Pythonic**: context managers, type hints, dataclasses, `Annotated`, exceptions.
- **Minimal dependencies**: `httpx` only.
- **Safe by default**: TLS verification, no redirects, no logging of values.

The goal is a small, well-typed package that lets a Python application go from a Yily token to a validated configuration object in a few lines, with the same behavior as the other official SDKs.
