# Módulos, Paquetes y Entornos

La gestión de dependencias y la organización del código son fundamentales en proyectos profesionales.

---

## Estructura de un proyecto profesional

```
mi_proyecto/
├── pyproject.toml        ← metadata + dependencias (reemplaza setup.py + requirements.txt)
├── .python-version       ← versión de Python (pyenv)
├── .env                  ← variables de entorno (nunca al repo)
├── .env.example          ← plantilla de variables (SÍ al repo)
├── README.md
│
├── src/
│   └── mi_proyecto/      ← código fuente (src layout)
│       ├── __init__.py
│       ├── main.py
│       ├── domain/
│       │   ├── __init__.py
│       │   └── product.py
│       ├── repository/
│       │   ├── __init__.py
│       │   └── postgres_repository.py
│       ├── usecase/
│       │   ├── __init__.py
│       │   └── product_usecase.py
│       └── api/
│           ├── __init__.py
│           └── routes.py
│
├── tests/
│   ├── conftest.py        ← fixtures globales de pytest
│   ├── unit/
│   └── integration/
│
└── scripts/              ← scripts de utilidad (migraciones, seeds, etc.)
    └── seed_db.py
```

---

## pyproject.toml — el estándar moderno

```toml
[project]
name = "mi-proyecto"
version = "0.1.0"
description = "API Backend con FastAPI"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110.0",
    "uvicorn[standard]>=0.27.0",
    "sqlalchemy>=2.0.0",
    "psycopg2-binary>=2.9.9",
    "pydantic>=2.0.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "httpx>=0.26.0",        # cliente async para tests de FastAPI
    "ruff>=0.3.0",          # linter + formatter
    "mypy>=1.8.0",          # type checker
]

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "UP", "S"]  # reglas a aplicar

[tool.mypy]
strict = true
python_version = "3.11"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing"
```

---

## Imports — buenas prácticas

```python
# Orden estándar (PEP 8):
# 1. Stdlib
# 2. Terceros
# 3. Propios (separados por línea en blanco)

import os
import sys
from pathlib import Path
from typing import Optional

import fastapi
from sqlalchemy.ext.asyncio import AsyncSession

from mi_proyecto.domain.product import Product
from mi_proyecto.repository.postgres_repository import PostgresProductRepository

# Imports relativos — dentro del mismo paquete
from .product import Product           # mismo directorio
from ..domain.product import Product   # directorio padre

# Alias para evitar conflictos
import numpy as np
import pandas as pd
from typing import Optional as Opt

# ¿Qué exporta un módulo? → __all__
# En domain/__init__.py
__all__ = ["Product", "ProductRepository", "ProductUseCase"]
```

---

## Variables de entorno — gestión correcta

```python
# .env (nunca commitear al repo)
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SECRET_KEY=super-secreto-cambiar-en-prod
DEBUG=false
LOG_LEVEL=info

# Usando pydantic-settings — validación de config
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )

    database_url: str
    secret_key: str
    debug: bool = False
    log_level: str = "info"
    allowed_origins: list[str] = ["http://localhost:3000"]

    # Validación custom
    @field_validator("log_level")
    @classmethod
    def validate_log_level(cls, v: str) -> str:
        valid = {"debug", "info", "warning", "error", "critical"}
        if v.lower() not in valid:
            raise ValueError(f"log_level inválido: {v}")
        return v.lower()

# Singleton — crear una sola vez
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Uso en la app
settings = get_settings()
print(settings.database_url)
```

---

## **init**.py — controlar la API pública del paquete

```python
# domain/__init__.py

# Importar lo que querés exponer
from .product import Product, ProductStatus
from .user import User, UserRole
from .exceptions import (
    AppError,
    NotFoundError,
    ValidationError,
    UnauthorizedError,
)

# __all__ define qué se exporta con "from domain import *"
__all__ = [
    "Product",
    "ProductStatus",
    "User",
    "UserRole",
    "AppError",
    "NotFoundError",
    "ValidationError",
    "UnauthorizedError",
]

# Ahora en lugar de:
# from mi_proyecto.domain.product import Product
# from mi_proyecto.domain.exceptions import NotFoundError

# Podés escribir:
# from mi_proyecto.domain import Product, NotFoundError
```

---

## Gestores de dependencias

### uv — el moderno (2024+)

```bash
# Instalar uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear proyecto nuevo
uv init mi-api
cd mi-api

# Añadir dependencias
uv add fastapi uvicorn sqlalchemy psycopg2-binary pydantic-settings

# Añadir dependencias de desarrollo
uv add --dev pytest pytest-asyncio ruff mypy httpx

# Ejecutar scripts
uv run python src/mi_proyecto/main.py
uv run pytest

# Sincronizar entorno (como npm install)
uv sync

# Actualizar todo
uv lock --upgrade
```

### pip + venv — el clásico

```bash
# Crear entorno virtual
python3 -m venv .venv

# Activar
source .venv/bin/activate      # Linux/Mac
.\.venv\Scripts\activate       # Windows

# Instalar
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Generar requirements.txt
pip freeze > requirements.txt

# Desactivar
deactivate
```

---

## Linting y formato — Ruff

Ruff reemplaza flake8, isort, y en parte black — todo en una herramienta, 10-100x más rápido.

```bash
uv add --dev ruff

# Verificar errores
ruff check src/

# Formatear código
ruff format src/

# En CI/CD
ruff check --output-format=github src/
ruff format --check src/  # falla si hay diferencias (sin modificar)
```

```toml
# pyproject.toml
[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8 naming
    "UP",  # pyupgrade — modernizar sintaxis
    "S",   # bandit — seguridad
    "B",   # bugbear — bugs comunes
    "SIM", # simplify
]
ignore = ["S101"]  # assert en tests está bien
```

## Mypy — type checking estático

```bash
uv add --dev mypy

mypy src/  # verificar tipos

# En pyproject.toml
[tool.mypy]
strict = true
```

Con `strict = true`, mypy verifica que:

- Todas las funciones tengan type hints
- No uses `Any` sin justificación
- Las variables no cambien de tipo inesperadamente
