# Observabilidad en Python — Logs, Métricas y Trazas

La guía de Go ya tiene `03-Observabilidad.md`. Este archivo cubre lo mismo pero para el stack Python (FastAPI + SQLAlchemy).

Los tres pilares de observabilidad:

- **Logs**: qué pasó
- **Métricas**: cuánto y qué tan seguido
- **Trazas**: cuánto tardó cada parte del flujo

---

## Logging estructurado con `structlog`

El logging estándar de Python emite texto plano. En producción necesitás JSON para que herramientas como Datadog, Loki o CloudWatch puedan parsear y buscar.

```bash
uv add structlog
```

```python
# logging_config.py
import logging
import sys
import structlog
from .config import settings

def setup_logging() -> None:
    """Configurar structlog para emitir JSON en producción, texto legible en dev."""

    shared_processors = [
        structlog.contextvars.merge_contextvars,       # contexto por request
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),   # timestamp ISO 8601
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if settings.environment == "production":
        # JSON en producción — parseable por herramientas externas
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]
    else:
        # Texto con colores en desarrollo
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True),
        ]

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    # También configurar el logging estándar para librerías (sqlalchemy, httpx, etc.)
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=logging.DEBUG if settings.debug else logging.INFO,
    )
```

### Usar structlog

```python
import structlog

logger = structlog.get_logger()

# Log simple
logger.info("servidor iniciado", port=8000, environment="production")

# Log con contexto
logger.info(
    "producto creado",
    product_id=product.id,
    product_name=product.name,
    user_id=current_user.id,
    duration_ms=elapsed_ms,
)

# Log de error con excepción
try:
    result = await repo.create(data)
except Exception as e:
    logger.error(
        "error al crear producto",
        error=str(e),
        product_name=data.name,
        exc_info=True,   # incluye stack trace
    )
    raise
```

### Contexto por request (request ID)

```python
# middleware/request_id.py
import uuid
import structlog
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())

        # Bindear el request_id al contexto de structlog para este request
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
        )

        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

# Registrar en main.py
app.add_middleware(RequestIDMiddleware)
```

Con esto, todos los logs de un request tienen automáticamente el `request_id`:

```json
{
  "event": "producto creado",
  "product_id": 42,
  "request_id": "abc-123",
  "method": "POST",
  "path": "/api/v1/products"
}
```

---

## Middleware de métricas de request

```python
# middleware/metrics.py
import time
import structlog
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

logger = structlog.get_logger()

class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()

        response = await call_next(request)

        duration_ms = (time.perf_counter() - start) * 1000

        # Loguear cada request con métricas
        log_fn = logger.warning if response.status_code >= 400 else logger.info
        log_fn(
            "request completado",
            method=request.method,
            path=request.url.path,
            status_code=response.status_code,
            duration_ms=round(duration_ms, 2),
            client_ip=request.client.host if request.client else None,
        )

        response.headers["X-Process-Time"] = f"{duration_ms:.2f}ms"
        return response
```

---

## Métricas con Prometheus

```bash
uv add prometheus-fastapi-instrumentator
```

```python
# main.py
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Expone /metrics automáticamente con métricas de todos los endpoints
Instrumentator().instrument(app).expose(app)

# Métricas personalizadas
from prometheus_client import Counter, Histogram, Gauge
import time

PRODUCTS_CREATED = Counter(
    "products_created_total",
    "Total de productos creados",
    ["category"]
)

DB_QUERY_DURATION = Histogram(
    "db_query_duration_seconds",
    "Duración de queries a la base de datos",
    ["operation", "table"],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0],
)

ACTIVE_SESSIONS = Gauge(
    "active_sessions",
    "Sesiones activas en este momento"
)

# Usar en el código
async def create_product(data: ProductCreate) -> Product:
    start = time.perf_counter()
    product = await repo.create(data)
    duration = time.perf_counter() - start

    # Registrar métricas
    PRODUCTS_CREATED.labels(category=data.category_id).inc()
    DB_QUERY_DURATION.labels(operation="insert", table="products").observe(duration)

    return product
```

---

## Healthcheck endpoint

```python
# api/routers/health.py
from fastapi import APIRouter
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends
import redis.asyncio as redis

router = APIRouter()

@router.get("/health")
async def health():
    """Health check básico — solo dice que la app está corriendo."""
    return {"status": "ok"}

@router.get("/health/ready")
async def readiness(session: AsyncSession = Depends(get_session)):
    """Readiness check — verifica que las dependencias están disponibles.
    Kubernetes usa esto para saber si enviar tráfico."""
    checks = {}

    # Verificar DB
    try:
        await session.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"

    # Verificar Redis
    try:
        rdb = get_redis_client()
        await rdb.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"

    all_ok = all(v == "ok" for v in checks.values())

    return JSONResponse(
        status_code=200 if all_ok else 503,
        content={"status": "ready" if all_ok else "not ready", "checks": checks}
    )
```

---

## Sentry — error monitoring en producción

```bash
uv add sentry-sdk[fastapi]
```

```python
# main.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn,
        environment=settings.environment,  # "production", "staging"
        integrations=[
            FastApiIntegration(transaction_style="endpoint"),
            SqlalchemyIntegration(),
        ],
        traces_sample_rate=0.1,   # 10% de requests trackeados (ajustar según volumen)
        send_default_pii=False,   # no enviar datos personales
    )
```

Con esto, cualquier excepción no manejada en producción aparece automáticamente en el dashboard de Sentry con el stacktrace completo, variables locales y el contexto del request.

---

## Leer logs en producción

```bash
# Logs de la app en Docker
docker logs mi-app --tail 100 -f

# Buscar errores
docker logs mi-app 2>&1 | grep '"level":"error"'

# Con jq para JSON logs
docker logs mi-app 2>&1 | jq 'select(.level == "error")'
docker logs mi-app 2>&1 | jq '{level, event, request_id, duration_ms}'

# systemd
journalctl -u mi-app -f                         # en tiempo real
journalctl -u mi-app --since "1 hour ago"        # última hora
journalctl -u mi-app -p err --since today        # solo errores de hoy
```

---

## Qué loguear (y qué no)

```python
# LOGUEAR — útil en producción
logger.info("usuario autenticado", user_id=user.id)
logger.info("producto creado", product_id=product.id, duration_ms=elapsed)
logger.warning("intento de login fallido", email=credentials.email, ip=client_ip)
logger.error("error al enviar email", user_id=user.id, exc_info=True)

# NO LOGUEAR — seguridad o ruido
logger.debug("password: %s", password)          # MAL — dato sensible
logger.info("token: %s", access_token)          # MAL — dato sensible
logger.debug("entrando a la función create")    # inútil — ruido
logger.info("request recibido")                  # el middleware ya lo logea
```
