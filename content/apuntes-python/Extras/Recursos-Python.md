# Recursos Python — Backend y SysAdmin

Recursos curados para seguir aprendiendo Python en contexto profesional de backend y administración de sistemas.

---

## Libros

| Libro                                                      | Por qué leerlo                                                                                                                     |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Fluent Python** (Luciano Ramalho, 2a ed. 2022)           | El libro definitivo de Python avanzado. Cubre el modelo de datos, decoradores, descriptores, concurrencia. Referencia obligatoria. |
| **Python Cookbook** (David Beazley & Brian Jones)          | Recetas para problemas reales. Excelente para aprender idioms de Python.                                                           |
| **Architecture Patterns with Python** (Percival & Gregory) | Clean Architecture, DDD y TDD aplicado a Python. Muy útil si ya usaste Clean Arch en Go.                                           |
| **Robust Python** (Patrick Viafore)                        | Type hints, dataclasses y escribir Python mantenible en equipos.                                                                   |

---

## Documentación oficial imprescindible

- **FastAPI**: https://fastapi.tiangolo.com — La documentación de FastAPI es excelente, empezar por el tutorial oficial
- **Pydantic v2**: https://docs.pydantic.dev — Especialmente la sección de validators y config
- **SQLAlchemy 2.0**: https://docs.sqlalchemy.org/en/20/ — Leer "ORM Quickstart" y "Unified Tutorial"
- **Alembic**: https://alembic.sqlalchemy.org — "Tutorial" oficial
- **asyncio**: https://docs.python.org/3/library/asyncio.html — Especialmente "Coroutines and Tasks"
- **pytest**: https://docs.pytest.org — "How to use fixtures"

---

## Sitios y blogs

| Recurso                   | Contenido                                                                |
| ------------------------- | ------------------------------------------------------------------------ |
| **realpython.com**        | Tutoriales profundos, bien escritos. Los mejores para temas intermedios. |
| **talkpython.fm**         | Podcast sobre Python, episodios con autores de librerías principales     |
| **pythonspeed.com**       | Performance y Docker para Python — muy práctico                          |
| **testdriven.io**         | FastAPI + Docker + testing — proyectos completos                         |
| **arjan.codes** (YouTube) | Clean Architecture con Python, muy práctico                              |
| **mCoding** (YouTube)     | Python avanzado, CPython internals                                       |

---

## Stack completo recomendado para nuevo proyecto

```toml
# pyproject.toml — stack probado en producción
[project]
dependencies = [
    # Web framework
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",

    # Validación y settings
    "pydantic>=2.9.0",
    "pydantic-settings>=2.6.0",

    # Base de datos
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",

    # Auth
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "python-multipart>=0.0.12",

    # HTTP client async
    "httpx>=0.28.0",

    # Utilidades
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
sysadmin = [
    "paramiko>=3.5.0",
    "fabric>=3.2.0",
    "psutil>=6.1.0",
    "typer[all]>=0.15.0",
    "schedule>=1.2.0",
]

dev = [
    # Testing
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "aiosqlite>=0.20.0",    # SQLite async para tests

    # Calidad de código
    "ruff>=0.8.0",
    "mypy>=1.13.0",

    # Type stubs
    "types-passlib",
    "types-paramiko",
]
```

---

## Comandos de referencia rápida

```bash
# Proyecto nuevo
mkdir mi_proyecto && cd mi_proyecto
uv init
uv add fastapi uvicorn[standard] sqlalchemy[asyncio] asyncpg alembic pydantic pydantic-settings

# Dev tools
uv add --dev pytest pytest-asyncio pytest-cov ruff mypy aiosqlite

# Servidor de desarrollo
uv run uvicorn src.mi_proyecto.main:app --reload --port 8000

# Tests
uv run pytest -v
uv run pytest --cov=src --cov-report=html

# Linting y formateo
uv run ruff check .         # lint
uv run ruff format .        # formatear
uv run ruff check --fix .   # auto-fix

# Type checking
uv run mypy src/

# Migraciones
uv run alembic revision --autogenerate -m "descripción"
uv run alembic upgrade head
uv run alembic downgrade -1
uv run alembic history

# Docker
docker compose up --build
docker compose exec app uv run pytest
docker compose exec app uv run alembic upgrade head
```

---

## Checklist proyecto Python en producción

```
Código:
[ ] Type hints en todo el código público
[ ] ruff sin errores
[ ] mypy sin errores (modo strict para código nuevo)
[ ] Coverage > 80% con pytest

Seguridad:
[ ] Variables sensibles en .env (nunca en código)
[ ] SECRET_KEY de mínimo 32 chars random
[ ] Passwords con bcrypt (passlib)
[ ] HTTPS configurado (nginx + certbot)
[ ] Rate limiting en endpoints de auth
[ ] CORS configurado correctamente
[ ] Headers de seguridad (fastapi-security-headers o nginx)

Base de datos:
[ ] Migraciones con Alembic (no AutoMigrate en producción)
[ ] Pool de conexiones configurado
[ ] Queries con parámetros (nunca string format en SQL)
[ ] Índices en columnas filtradas/ordenadas frecuentemente
[ ] Backups automáticos configurados

Infraestructura:
[ ] Dockerfile multi-stage
[ ] Usuario non-root en el container
[ ] Health check configurado
[ ] Logs estructurados (no print)
[ ] Métricas de aplicación (optional: prometheus)

Deploy:
[ ] CI/CD con tests automáticos
[ ] Migraciones como paso separado (no al iniciar la app)
[ ] Variables de entorno en secreto manager (no en .env en servidor)
[ ] Monitoreo de errores (Sentry o similar)
```

---

## Python vs Go — cuándo usar cada uno

| Tarea                                | Recomendación                                |
| ------------------------------------ | -------------------------------------------- |
| API REST alta performance            | Go (gin/echo)                                |
| API REST con validación compleja     | Python (FastAPI + Pydantic)                  |
| Scripts de automatización            | Python                                       |
| Admin de servidores (SSH, scripting) | Python (fabric/paramiko)                     |
| CLI tools complejos                  | Go o Python (typer)                          |
| Procesamiento de datos, ML           | Python (pandas, numpy, scikit-learn)         |
| Microservicios con alta concurrencia | Go                                           |
| Prototipado rápido                   | Python                                       |
| Systemd services largos              | Cualquiera (Go más adecuado para eficiencia) |
