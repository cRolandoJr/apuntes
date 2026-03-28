# Docker para Python

Docker para Python tiene algunas particularidades: los multi-stage builds son esenciales para reducir el tamaño de la imagen, y `uv` es la herramienta ideal para instalar dependencias en Docker de forma reproducible y rápida.

---

## Dockerfile multi-stage con `uv`

```dockerfile
# syntax=docker/dockerfile:1

# ─── Etapa 1: Builder ────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

# Instalar uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copiar archivos de dependencias primero (mejor cache)
COPY pyproject.toml uv.lock ./

# Instalar dependencias en /app/.venv (sin dev dependencies)
RUN uv sync --frozen --no-dev --no-editable

# Copiar código fuente
COPY src/ ./src/

# ─── Etapa 2: Runtime ────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

# Usuario no-root para seguridad
RUN groupadd --gid 1001 appuser && \
    useradd --uid 1001 --gid appuser --shell /bin/bash --create-home appuser

WORKDIR /app

# Copiar solo lo necesario del builder
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src

# Activar el virtualenv
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app/src

USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"

CMD ["uvicorn", "mi_proyecto.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

---

## `.dockerignore`

```dockerignore
# Control de versiones
.git
.gitignore

# Entornos virtuales
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.Python

# Tests y herramientas de desarrollo
tests/
.pytest_cache/
.mypy_cache/
.ruff_cache/
htmlcov/
.coverage

# Documentación
docs/
*.md
!README.md

# Variables de entorno
.env
.env.*
!.env.example

# Editor
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Docker
Dockerfile*
docker-compose*.yml
```

---

## `docker-compose.yml` para desarrollo

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: builder # usar la etapa builder para desarrollo
    volumes:
      - ./src:/app/src # hot reload del código
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:password@db:5432/mi_db
      - SECRET_KEY=dev-secret-key-no-usar-en-produccion
      - DEBUG=true
    command: >
      uvicorn mi_proyecto.main:app
      --host 0.0.0.0
      --port 8000
      --reload
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mi_db
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  pgdata:
```

---

## `docker-compose.override.yml` — solo para desarrollo local

```yaml
# docker-compose.override.yml (no commitear variables sensibles)
services:
  app:
    environment:
      - LOG_LEVEL=DEBUG
    volumes:
      - ./tests:/app/tests # para correr tests dentro del container
```

---

## Scripts de entrypoint

```bash
#!/bin/bash
# entrypoint.sh
set -e  # salir si cualquier comando falla

echo "Esperando a que la DB esté lista..."
until python -c "
import asyncio, asyncpg, os
async def check():
    conn = await asyncpg.connect(os.environ['DATABASE_URL'].replace('postgresql+asyncpg', 'postgresql'))
    await conn.close()
asyncio.run(check())
" 2>/dev/null; do
  echo "DB no disponible, reintentando en 2 segundos..."
  sleep 2
done

echo "Aplicando migraciones de Alembic..."
uv run alembic upgrade head

echo "Iniciando aplicación..."
exec "$@"
```

```dockerfile
# En el Dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["uvicorn", "mi_proyecto.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Comandos útiles

```bash
# Build
docker build -t mi-app:latest .
docker build -t mi-app:latest --target runtime .  # solo etapa runtime

# Verificar tamaño de imagen
docker images mi-app

# Correr en local
docker run -p 8000:8000 --env-file .env mi-app:latest

# Development con compose
docker compose up --build          # construir y levantar
docker compose up -d               # en background
docker compose logs -f app         # ver logs
docker compose exec app bash       # entrar al container
docker compose exec app uv run pytest  # correr tests

# Aplicar migraciones en running container
docker compose exec app uv run alembic upgrade head

# Limpiar
docker compose down -v             # bajar y eliminar volúmenes (borra DB!)
docker compose down                # bajar sin eliminar volúmenes
```

---

## Variables de entorno en producción

```bash
# Nunca usar .env en producción — usar variables del sistema o secretos
# En Kubernetes: Secrets
# En cloud: AWS Secrets Manager, GCP Secret Manager, etc.
# En servidores bare metal: systemd EnvironmentFile

# /etc/mi-app/env (permisos 600, propietario el usuario de la app)
DATABASE_URL=postgresql+asyncpg://user:password@db-server:5432/mi_db
SECRET_KEY=super-long-random-key-here

# /etc/systemd/system/mi-app.service
[Service]
EnvironmentFile=/etc/mi-app/env
ExecStart=/usr/bin/docker run --env-file /etc/mi-app/env mi-app:latest
```

---

## Imagen de producción optimizada

```dockerfile
# Para producción: imagen más pequeña con distroless o slim
FROM gcr.io/distroless/python3-debian12 AS production

COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

USER nonroot:nonroot
EXPOSE 8000

ENTRYPOINT ["uvicorn", "mi_proyecto.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Comparación de tamaños típicos

| Base image             | Tamaño aproximado                     |
| ---------------------- | ------------------------------------- |
| python:3.12            | ~1 GB                                 |
| python:3.12-slim       | ~150 MB                               |
| python:3.12-alpine     | ~60 MB (problemas con algunas libs C) |
| distroless/python3     | ~50 MB                                |
| Con multi-stage + slim | ~200 MB (app incluida)                |

**Recomendación**: `python:3.12-slim` para la etapa runtime es el mejor balance entre tamaño y compatibilidad.
