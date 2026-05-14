---
id: t_T8xY1c
slug: fastapi-skeleton
title: FastAPI app factory + Settings + health/metrics endpoints + request-logging middleware
workgraph: foundation
order: 3
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t3-fastapi-skeleton 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T3-1 — `src/vfobs/config.py` defines `Settings(BaseSettings)`
    (pydantic-settings) with at least: `database_url: PostgresDsn`,
    `ingest_token: SecretStr` (consumed in T4), `log_level: str =
    "INFO"`, `service_name: str = "vfobs"`. Env-prefix `VFOBS_`
    (e.g., `VFOBS_DATABASE_URL`). Exposes `get_settings() ->
    Settings` as an `lru_cache`-wrapped factory (Singleton pattern).
  - AC-T3-2 — `src/vfobs/main.py` defines `create_app(settings:
    Settings | None = None) -> FastAPI`. App factory pattern —
    callers (tests, ASGI server) pass a Settings or default. Mounts
    health, metrics, and the request-logging middleware. Returns
    a configured `FastAPI` instance.
  - AC-T3-3 — `src/vfobs/api/health.py` defines two endpoints:
    `GET /healthz` always returns `{"status": "ok"}` with 200
    (liveness — process alive). `GET /readyz` runs a trivial query
    against the DB (`SELECT 1`) and returns
    `{"status": "ok", "db": "ok"}` 200 on success;
    `{"status": "not_ready", "db": "<error class name>"}` 503 on
    failure. DB engine is fetched from app state (set in
    `create_app`'s lifespan).
  - AC-T3-4 — `src/vfobs/api/metrics.py` exposes
    `GET /metrics` serving Prometheus exposition format via
    `prometheus_client.make_asgi_app()`. Registers two metrics in
    `src/vfobs/observability/metrics_registry.py`:
    `vfobs_http_requests_total{method,path,status}` (Counter) and
    `vfobs_http_request_duration_seconds{method,path}` (Histogram).
    These are populated by the logging-middleware Decorator from
    AC-T3-5.
  - AC-T3-5 — `src/vfobs/middleware/logging.py` defines
    `RequestLoggingMiddleware` (ASGI middleware Decorator). For
    each request: generates a `request_id` UUID, attaches it to
    a `contextvars.ContextVar` so log records pick it up,
    increments the Counter from AC-T3-4 with method+path+status,
    observes the Histogram with duration, and emits one structured
    log line at INFO with method, path, status, duration_ms,
    request_id.
  - AC-T3-6 — Unit: `tests/unit/test_health.py` constructs an app
    via `create_app(settings=...)` with a mocked engine (the
    `/readyz` test installs a sync stub that succeeds; a second
    test installs one that raises). Asserts response shapes,
    status codes, and that `/healthz` doesn't touch the DB.
  - AC-T3-7 — Unit: `tests/unit/test_middleware_logging.py`
    constructs the middleware with stub upstream, fires a fake
    request, asserts the counter incremented, histogram observed,
    and a log line emitted with the expected fields (captured via
    `caplog`). Uses `contextvars` correctly under asyncio.
  - AC-T3-8 — Integration: `tests/integration/test_app_lifecycle.py`
    spins the app under `httpx.AsyncClient` (no real network),
    hits `/healthz` and `/readyz` and `/metrics`, asserts shapes +
    content-types. Uses a disposable Postgres for `/readyz`'s real
    query (reuses the T0 fixture pattern).
  - AC-T3-9 — Branch `wg-vfobs-foundation/t3-fastapi-skeleton` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `src/vfobs/config.py` (new — extends WG0 placeholder if any).
- `src/vfobs/main.py` (new).
- `src/vfobs/api/__init__.py` (new).
- `src/vfobs/api/health.py` (new).
- `src/vfobs/api/metrics.py` (new).
- `src/vfobs/observability/__init__.py` (new).
- `src/vfobs/observability/metrics_registry.py` (new) — Prometheus
  Counter + Histogram definitions kept out of the middleware file
  so they are importable without side effects.
- `src/vfobs/middleware/__init__.py` (new).
- `src/vfobs/middleware/logging.py` (new) — ASGI middleware +
  `request_id` ContextVar.
- `tests/unit/test_health.py` (new).
- `tests/unit/test_middleware_logging.py` (new).
- `tests/integration/test_app_lifecycle.py` (new).
- `pyproject.toml` (modify) — confirm `prometheus-client>=0.20,<1.0`
  is in `[project] dependencies`.

(NOT touched: `src/vfobs/events/`, `src/vfobs/repositories/`,
`src/vfobs/api/events.py` (that's T4), `alembic/`, `charts/`.)

## Implementation sketch

**Patterns: Singleton (`get_settings()` LRU-cached factory) +
Decorator (request-logging middleware wraps each request) + App
Factory (`create_app(settings)` returns a configured FastAPI; same
shape as vtaskforge's app factory).**

### config.py

```python
from functools import lru_cache
from pydantic import PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="VFOBS_",
        env_file=".env",
        extra="ignore",
    )

    database_url: PostgresDsn
    ingest_token: SecretStr
    log_level: str = "INFO"
    service_name: str = "vfobs"

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()  # type: ignore[call-arg]  # env-resolved
```

### main.py

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import AsyncEngine
from vfobs.config import Settings, get_settings
from vfobs.db import build_engine
from vfobs.api import health, metrics
from vfobs.middleware.logging import RequestLoggingMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    engine: AsyncEngine = build_engine(app.state.settings)
    app.state.engine = engine
    try:
        yield
    finally:
        await engine.dispose()

def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or get_settings()
    app = FastAPI(title="vfobs", version="0.0.1", lifespan=lifespan)
    app.state.settings = settings
    app.add_middleware(RequestLoggingMiddleware)
    app.include_router(health.router)
    app.mount("/metrics", metrics.metrics_app)
    return app
```

### health.py

```python
from fastapi import APIRouter, Request, Response
from sqlalchemy import text

router = APIRouter()

@router.get("/healthz")
async def healthz() -> dict:
    return {"status": "ok"}

@router.get("/readyz")
async def readyz(request: Request) -> Response:
    engine = request.app.state.engine
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
    except Exception as exc:
        return JSONResponse(
            status_code=503,
            content={"status": "not_ready", "db": exc.__class__.__name__},
        )
    return JSONResponse(content={"status": "ok", "db": "ok"})
```

### metrics_registry.py

```python
from prometheus_client import Counter, Histogram, CollectorRegistry

# A dedicated registry keeps the test scope clean.
REGISTRY = CollectorRegistry(auto_describe=True)

REQUESTS = Counter(
    "vfobs_http_requests_total",
    "Total HTTP requests handled",
    ["method", "path", "status"],
    registry=REGISTRY,
)

DURATION = Histogram(
    "vfobs_http_request_duration_seconds",
    "HTTP request duration",
    ["method", "path"],
    registry=REGISTRY,
)
```

### middleware/logging.py

```python
import contextvars
import logging
import time
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from vfobs.observability.metrics_registry import REQUESTS, DURATION

request_id_ctx: contextvars.ContextVar[str] = contextvars.ContextVar(
    "request_id", default="-"
)

logger = logging.getLogger("vfobs.request")

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        rid = str(uuid.uuid4())
        request_id_ctx.set(rid)
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start
        path = request.url.path
        method = request.method
        REQUESTS.labels(method=method, path=path, status=response.status_code).inc()
        DURATION.labels(method=method, path=path).observe(duration)
        logger.info(
            "request",
            extra={
                "method": method,
                "path": path,
                "status": response.status_code,
                "duration_ms": int(duration * 1000),
                "request_id": rid,
            },
        )
        return response
```

## Fail-loud directive (R3)

If any required step (test scaffolding for the lifespan-mounted
engine, prometheus_client integration, branch push) cannot be
completed due to missing dependencies, missing tools, or blocked
external constraints: report the failure explicitly in your
completion notes — do not rationalize partial completion as
success. The judge will fail tasks that report dishonest success;
tasks that report failure honestly can be reworked.

# Constraints

- `get_settings()` MUST be the only constructor for Settings used
  outside tests. Tests construct `Settings(database_url=..., ingest_token=...)`
  directly to inject test config. The lru_cache means runtime
  Settings is a Singleton.
- `/healthz` MUST NOT touch the DB. If it does, k8s liveness probes
  will fire pod restarts during DB blips — wrong behavior.
- `/readyz` MUST return 503 on DB failure (not 200 + error in body).
  k8s readiness probes need an HTTP status to act on.
- The middleware MUST work with FastAPI's `lifespan` + async stack
  (no sync calls from inside async handler).
- The metrics registry MUST be importable without side-effects
  (separate file from the middleware; tests import REGISTRY and
  reset between tests if needed).

# Out of scope

- Authentication middleware. T4 adds auth as a FastAPI dependency
  on POST /events specifically — `/healthz`, `/readyz`, `/metrics`
  are intentionally unauthenticated (cluster-internal endpoints).
- OpenTelemetry tracing instrumentation. v2 candidate.
- Sentry / error tracking integration. v2 candidate.
- Custom JSON log formatter / structured-log dispatcher.
  Stdlib `logging` with `extra=...` is good enough for v1; ops
  can pipe through a JSON formatter at deploy time.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §F, NFR4
- FastAPI lifespan docs: https://fastapi.tiangolo.com/advanced/events/
- pydantic-settings docs: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- prometheus_client ASGI example:
  https://prometheus.github.io/client_python/exporting/http/
- vtaskforge `src/vtaskforge/` for app-factory pattern reference
