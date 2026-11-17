# Yily Python SDK — Architecture Proposal

## 1. Objective

Introduce a lightweight, fully typed Python SDK that lets applications retrieve configuration variables from Yily and use them directly in Python code: scripts, web services (FastAPI, Django, Flask), workers and CI jobs.

The SDK follows Yily's philosophy — **simple and effective** — and stays aligned with the accepted [Go SDK proposal](go.md): same scope, same endpoints, same error semantics, adapted to Python conventions.

## 2. Scope

The SDK supports:

- Connecting to a Yily instance and selecting an environment.
- Authenticating with a pre-provisioned Personal Access Token (PAT).
- Retrieving every variable of an environment.
- Retrieving individual variables by name, optionally at a specific version.
- Loading values into a typed configuration object (dataclass or Pydantic model), with automatic type conversion.
- Optionally exporting values into `os.environ` for libraries that only read environment variables.
- Synchronous **and** asynchronous usage with the same API.

Out of scope for the first release, as in the Go SDK: project, environment, tag and token administration. Token creation remains session-only on the server and is never exposed through a PAT.

Write operations (creating variables, adding versions) are discussed as an optional second phase in [section 10](#10-managing-secrets-phase-2), so the first release can ship as a small read-only package.

## 3. Project Structure

Like the Go SDK, the Python SDK lives in a dedicated repository: `github.com/yannick2009/yily-sdk-python`.

```text
.
├── pyproject.toml
├── README.md
├── CHANGELOG.md
├── src/
│   └── yily/
│       ├── __init__.py        # Public exports
│       ├── py.typed           # PEP 561 marker: the package ships type hints
│       ├── _config.py         # Config object, validation and from_env()
│       ├── _client.py         # Client (synchronous)
│       ├── _async_client.py   # AsyncClient (asynchronous)
│       ├── _http.py           # Shared request building, envelope parsing, retries
│       ├── _loader.py         # Typed loading into dataclasses / Pydantic models
│       ├── _convert.py        # String → Python type conversion
│       ├── _secret.py         # Secret wrapper that masks values in repr/logs
│       └── errors.py          # Exception hierarchy
├── tests/
│   ├── unit/
│   └── integration/
└── examples/
    ├── basic.py
    ├── dataclass_config.py
    ├── fastapi_app.py
    └── django_settings.py
```

The distribution is published on PyPI as `yily` (or `yily-sdk` if the name is taken) and imported as `import yily`. Every module except `errors` is private (underscore prefix); the public surface is what `yily/__init__.py` re-exports.

## 4. Public API

| Symbol | Purpose |
|---|---|
| `yily.Client(...)` | Synchronous client. Usable as a context manager. |
| `yily.AsyncClient(...)` | Asynchronous client with the same methods, as coroutines. |
| `client.get(name, *, version=None) -> str` | Value of one variable (active version by default). |
| `client.get_many(selection) -> dict[str, str]` | Several variables in one request; `selection` is a list of names or a `{name: version}` mapping. |
| `client.get_all() -> dict[str, str]` | Every variable of the environment with its active value. |
| `client.load(model) -> model` | Builds a typed configuration object from a dataclass or Pydantic model. |
| `client.export_to_environ(*, overwrite=False) -> list[str]` | Copies values into `os.environ`; returns the names that were set. |
| `client.token_info() -> TokenInfo` | Metadata of the current PAT (name, scope, expiry), via `GET /tokens/me`. |
| `yily.Var(name, *, version=None)` | Field annotation used by `load()` to map a field to a variable and version. |
| `yily.Secret` | String wrapper whose `repr()`/`str()` are masked; `.reveal()` returns the value. |

Minimal example:

```python
import yily

with yily.Client(
    base_url="https://yily.example.com",
    environment_id="ENVx9boixbYI9by",
    token="yily_...",
) as client:
    db_host = client.get("DB_HOST")
    old_key = client.get("API_KEY", version=2)
    everything = client.get_all()
```

## 5. Authentication and Configuration

### 5.1. Personal Access Tokens

The SDK authenticates every request with:

```http
Authorization: Bearer yily_<token>
```

PATs are created in the web interface, which is the only place where tokens can be managed. The recommended scope for applications is **read**. `token_info()` lets an application check the scope and expiry of its token, for example in a health check, without the SDK making that extra request on every startup.

The token is never logged, never included in exception messages and is excluded from `repr(client)` / `repr(config)`.

### 5.2. Configuration

```python
client = yily.Client(
    base_url="https://yily.example.com",  # required, http(s) only
    environment_id="ENVx9boixbYI9by",      # required, validated format (ENV + 12 base62 chars)
    token="yily_...",                      # required
    timeout=5.0,                           # seconds, default 5 (same as Go SDK)
    max_retries=2,                         # retries for transient failures, default 2
    verify=True,                           # TLS verification, or path to a CA bundle
    http_client=None,                      # optional pre-configured httpx.Client / AsyncClient
)
```

Validation happens in the constructor and raises `yily.ConfigurationError` immediately (bad URL scheme, empty token, malformed environment ID). Trailing slashes and `/api/v1` suffixes in `base_url` are normalized.

Unlike the Go SDK, which leaves environment loading to the caller, Python users commonly configure services through environment variables, so the SDK offers an **explicit** opt-in constructor:

```python
client = yily.Client.from_env()  # reads YILY_URL, YILY_TOKEN, YILY_ENVIRONMENT_ID
```

Nothing is read from the environment unless `from_env()` is called. Keyword arguments passed to `from_env()` override environment values.

### 5.3. Transport security

A warning is emitted when `base_url` uses plain `http://` with a host other than `localhost`/`127.0.0.1`, since PATs and secret values would travel unencrypted.

Redirects are **not followed** by default (`follow_redirects=False` in `httpx`). A Yily API never needs to redirect a client, and following a redirect could send the `Authorization` header to another host — for example after a misconfigured reverse proxy redirects `http://` to a different domain. A `3xx` response raises an `APIError` whose message includes the `Location` target, so the user can fix `base_url` instead of silently leaking the token.

## 6. Retrieving Secrets

### 6.1. Batch endpoint

`get`, `get_many` and `load` use the read-only batch endpoint already used by the Go SDK:

```http
POST /api/v1/environments/{environmentID}/variables/values
Content-Type: application/json
Authorization: Bearer yily_<token>

{
  "variables": [
    { "name": "DB_HOST", "version": 2 },
    { "name": "DB_PORT" }
  ]
}
```

Response (standard Yily envelope):

```json
{
  "status": 200,
  "message": "Variable values retrieved successfully",
  "data": { "DB_HOST": "localhost", "DB_PORT": "5432" }
}
```

SDK behavior, matching the server contract:

- An omitted version selects the active version.
- A requested name absent from `data` raises `yily.VariableNotFoundError` (with the name, never a value).
- A variable without an active version is returned as an empty string.
- The server accepts 1 to 1,000 selections per request; the SDK transparently splits larger selections into several requests.
- Duplicate names with different versions in one call raise `yily.ConflictingVersionError` locally, before any request is sent.

### 6.2. Listing every variable

`get_all()` first lists the environment with `GET /api/v1/environments/{environmentID}/variables`, then fetches every value through the batch endpoint — the same two-step strategy as the Go SDK.

### 6.3. Synchronous and asynchronous clients

Both clients share the same request-building, parsing and error-mapping code in `_http.py`; only the I/O layer differs. This keeps the two APIs strictly identical:

```python
async with yily.AsyncClient.from_env() as client:
    values = await client.get_many(["DB_HOST", "DB_PORT"])
```

## 7. Loading Typed Configuration

### 7.1. Declaring a configuration

`load()` accepts a **dataclass** (standard library, no extra dependency) or a **Pydantic v2 model** (when Pydantic is installed). Variable names default to the field name in upper case; `yily.Var` overrides the name and/or selects a version through `typing.Annotated`:

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Annotated

import yily


@dataclass(frozen=True)
class AppConfig:
    db_host: str                                        # → DB_HOST, active version
    db_port: int                                        # → DB_PORT, converted to int
    jwt_secret: Annotated[yily.Secret, yily.Var("JWT_SECRET")]
    legacy_key: Annotated[str, yily.Var("API_KEY", version=2)]
    debug: bool = False                                 # → DEBUG, default used if missing
    sentry_dsn: str | None = None                       # optional: None if missing


config = client.load(AppConfig)
```

### 7.2. Loading rules

The loader:

1. Inspects the fields and their type hints.
2. Collects variable names and versions.
3. Retrieves every value in **one** batch request (split only above 1,000 variables).
4. Converts each value to the annotated type.
5. Builds the object only if every conversion succeeds; otherwise raises a single `yily.LoadError` listing every failing field.

Missing variables:

- A field with a default value (or typed `X | None`) uses its default.
- A required field raises `yily.LoadError` listing the missing variable names.

Mapping the same variable to two fields with different versions raises `yily.ConflictingVersionError`, mirroring `ErrConflictingVariableVersion` in Go.

### 7.3. Supported types

| Annotation | Conversion |
|---|---|
| `str` | as is |
| `int`, `float`, `decimal.Decimal` | standard constructors |
| `bool` | `true/false`, `1/0`, `yes/no`, `on/off` (case-insensitive); anything else is an error |
| `datetime.timedelta` | Go-style durations (`30s`, `5m`, `1h30m`), for parity with `time.Duration` |
| `pathlib.Path` | `Path(value)` |
| `list[T]` | comma-separated values, each converted to `T` |
| `yily.Secret` | wrapped, masked in `repr()` |
| `X \| None` | optional field |

With Pydantic models, the SDK passes raw strings to `model_validate()` and lets Pydantic perform conversion and validation, so users keep all Pydantic features (`SecretStr`, validators, constraints).

Conversion errors include the variable name and the target type, **never the value**.

### 7.4. Exporting to `os.environ`

Many libraries only read `os.environ`. For them:

```python
client.export_to_environ()                     # fills missing keys only
client.export_to_environ(overwrite=True)       # replaces existing keys
```

## 8. Error Handling

### 8.1. Exception hierarchy

```text
YilyError                          # base class for everything raised by the SDK
├── ConfigurationError             # invalid client configuration
├── ConflictingVersionError        # same variable requested at two versions
├── LoadError                      # load() failed (missing variables, conversions)
│   └── ConversionError            # one value could not be converted
├── TransportError                 # network failure, DNS, TLS
│   └── TimeoutError
└── APIError                       # non-2xx response; exposes .status, .code, .message
    ├── AuthenticationError        # 401: invalid, expired or revoked token
    ├── PermissionDeniedError      # 403: e.g. TOKEN_WRITE_FORBIDDEN
    ├── NotFoundError              # 404
    │   ├── EnvironmentNotFoundError
    │   ├── VariableNotFoundError       # also raised for names missing from a batch response
    │   └── VariableVersionNotFoundError
    ├── BadRequestError            # 400: VALIDATION_FAILED, VARIABLE_SELECTION_DUPLICATE, …
    ├── RateLimitError             # 429
    └── ServerError                # 5xx
```

### 8.2. Mapping server responses

Every API error carries the server's stable `code` (e.g. `VARIABLE_NOT_FOUND`, `TOKEN_EXPIRED`, `ENVIRONMENT_NOT_FOUND`). The SDK maps the code to the most specific exception first, then falls back to the HTTP status: some responses, such as the authentication middleware's 401, do not include a code. Unknown codes produce the generic class for their status, so new server codes never break existing clients.

```python
try:
    config = client.load(AppConfig)
except yily.AuthenticationError:
    sys.exit("Yily token is invalid, expired or revoked")
except yily.LoadError as exc:
    sys.exit(f"Missing or invalid configuration: {exc.fields}")
except yily.YilyError as exc:
    sys.exit(f"Could not reach Yily: {exc}")
```

### 8.3. Retries and timeouts

- Default timeout: 5 seconds per request.
- Retries (`max_retries`, default 2) with exponential backoff and jitter, only for connection errors, timeouts, `429`, `502`, `503` and `504`. All read operations, including the batch `POST`, are non-mutating and therefore safe to retry. A `Retry-After` header is honored when present.
- `401`, `403`, `404` and `400` are never retried.

## 9. Security

- Secret values are returned to the caller and never logged, cached on disk or included in exceptions.
- `yily.Secret` is available for fields that should not leak through `repr()`, `print()` or framework debug pages.
- The SDK does not log at all by default. It uses the standard `logging` module under the `yily` logger, at `DEBUG` level, for request method, path, status and duration only — never headers, bodies or values.
- TLS verification is enabled by default and can only be relaxed explicitly (`verify=False` or a custom CA bundle).
- Redirects are never followed, so the PAT cannot be forwarded to another host (see [5.3](#53-transport-security)).
- Expired or revoked tokens (`TOKEN_EXPIRED`, `TOKEN_REVOKED`) raise `AuthenticationError` immediately and are never retried. A password recovery on the server revokes every PAT, so applications must be ready to receive this error.
- No in-memory cache is kept in the first release, so a rotated secret is visible on the next read. A time-based cache can be added later if users ask for it.

## 10. Managing Secrets (phase 2)

The issue mentions managing secrets as well as retrieving them. To keep the first release small and consistent with the Go SDK, write operations are proposed as a **separate, optional phase**, available only with a **write-scoped** PAT:

```python
client.set("API_KEY", "new-value")    # creates the variable or adds a new version
client.delete("OLD_KEY")
```

These map to existing write routes (`POST /environments/{id}/variables`, `POST /variables/{id}/versions`, `DELETE /variables/{id}`) after resolving the name to an ID through the list endpoint's `search` filter. Using them with a read token raises `PermissionDeniedError` (`TOKEN_WRITE_FORBIDDEN`). Maintainers can decide whether this phase belongs in the SDK at all or stays in the CLI.

## 11. Testing

- **Unit tests** with `pytest` and [`respx`](https://lundberg.github.io/respx/) to mock `httpx` requests: configuration validation, envelope parsing, error mapping for every documented code, batch splitting, version conflicts, type conversion, missing values, retries and timeouts.
- **Async tests** run the same scenarios against `AsyncClient` through a shared parametrized test suite, ensuring both clients behave identically.
- **Integration tests** in CI against a real Yily server started from the official Docker image (`deploy/docker`), with a read token provisioned by a setup script.
- **Static checks**: `mypy --strict`, `ruff` (lint and format).
- **CI matrix**: Python 3.10 to 3.13 on Linux, with one macOS and one Windows job.

## 12. Packaging and Dependencies

- **Python**: 3.10+ (3.9 has reached end of life).
- **Runtime dependency**: [`httpx`](https://www.python-httpx.org/) only. It provides sync and async clients behind a single API, timeouts, HTTP/2 and good testability. The standard library offers no async HTTP client, which would force two separate implementations.
- **Optional extra**: `pip install "yily[pydantic]"` enables Pydantic model loading. Pydantic is imported lazily, so it is never required.
- **Build**: `pyproject.toml` with `hatchling`, `src/` layout, `py.typed` marker; development with `uv`.
- **Release**: semantic versioning, changelog, PyPI publication from GitHub Actions using Trusted Publishing (no long-lived PyPI token).
- **Compatibility**: the SDK targets `/api/v1`. Its README documents the minimum Yily server version (the one introducing the batch endpoint and PAT scopes).

## 13. Usage Examples

### 13.1. FastAPI

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import yily

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with yily.AsyncClient.from_env() as client:
        app.state.config = await client.load(AppConfig)
    yield

app = FastAPI(lifespan=lifespan)
```

### 13.2. Django `settings.py`

```python
import yily

with yily.Client.from_env() as _yily:
    _values = _yily.get_many(["SECRET_KEY", "DATABASE_URL"])

SECRET_KEY = _values["SECRET_KEY"]
DATABASE_URL = _values["DATABASE_URL"]
```

### 13.3. Scripts and CI

```python
import subprocess
import yily

with yily.Client.from_env() as client:
    client.export_to_environ()

subprocess.run(["./deploy.sh"], check=True)  # sees the variables in its environment
```

## 14. Documentation Plan

The SDK repository ships a README for a quick start, plus a documentation site generated with MkDocs (Material theme) and `mkdocstrings`, which builds the API reference from docstrings and type hints. All examples in the docs are executed in CI against the mocked server, so they cannot drift from the code.

| Page | Content |
|---|---|
| Getting started | Installation, creating a **read** PAT in the Yily web interface, finding the environment ID, first `get()` call. |
| Configuration | Every constructor option, `from_env()` and its variables, timeouts, retries, TLS and custom CA bundles. |
| Retrieving variables | `get`, `get_many`, `get_all`, version selection, behavior for missing and empty values. |
| Typed configuration | `load()` with dataclasses and Pydantic, `yily.Var`, `yily.Secret`, supported types, error reporting. |
| Async usage | `AsyncClient`, sharing one client across an application, closing it properly. |
| Error handling | The exception hierarchy, server error codes and their mapping, what is retried. |
| Framework guides | FastAPI, Django, Flask, Celery workers. |
| CI/CD | GitHub Actions and GitLab CI examples: storing the PAT as a CI secret, `export_to_environ()` before running a deployment. |
| Security best practices | Read-scoped tokens, token expiry and rotation, HTTPS, never logging values, behavior after a password recovery. |
| Migrating from `.env` files | Replacing `python-dotenv` with Yily step by step, using Yily's `.env` import to move existing variables. |
| Changelog and compatibility | SDK versions and the minimum Yily server version each one supports. |

## 15. Roadmap

Each phase ends with a usable, tagged release, so feedback can start early. Phases 1 to 4 cover the scope approved for the Go SDK; phase 5 depends on the maintainers' decision in [section 10](#10-managing-secrets-phase-2).

| Phase | Content | Release |
|---|---|---|
| **1. Foundation** | Repository setup (`pyproject.toml`, `src/` layout, `ruff`, `mypy --strict`, CI matrix), `Config` and validation, shared HTTP layer (envelope parsing, error mapping, no redirects), exception hierarchy. | — |
| **2. Read operations** | Synchronous `Client`: `get`, `get_many` (batch splitting, version conflicts), `get_all`, `token_info`. Unit tests with `respx`. | `0.1.0` |
| **3. Typed loading and async** | `load()` for dataclasses, type conversion, `yily.Var`, `yily.Secret`, `export_to_environ()`, optional Pydantic support, `AsyncClient` with the shared test suite, retries and backoff. | `0.2.0` |
| **4. Stabilization and release** | Integration tests against the Yily Docker image, documentation site, framework examples, PyPI Trusted Publishing, API freeze. | `1.0.0` |
| **5. Write operations (optional)** | `set` and `delete` with write-scoped PATs, if approved. | `1.1.0` |

Out of the roadmap unless users ask for it: an in-memory cache with TTL, a CLI (Yily already ships `yily-cli`), and project, environment or tag administration.

## 16. Design Principles

- **Simplicity**: a handful of methods covering the common cases.
- **Parity with Go**: same scope, endpoints and error semantics, so the SDKs stay consistent.
- **Pythonic**: context managers, type hints, dataclasses, `Annotated`, exceptions instead of return codes.
- **Minimal dependencies**: `httpx` only; Pydantic optional.
- **Read-first and least privilege**: read-scoped tokens by default, writes only as an explicit phase 2.
- **Safe by default**: TLS on, no logging of values, masked secrets.

The goal is a small, well-typed package that lets a Python application go from a Yily token to a validated configuration object in a few lines.
