# Alembic — Migraciones de Base de Datos

Alembic es el sistema de migraciones para SQLAlchemy. Similar a `golang-migrate` en Go pero genera las migraciones automáticamente a partir de los cambios en los modelos.

---

## Instalación y configuración inicial

```bash
uv add alembic
uv run alembic init alembic
```

Esto crea:

```
alembic/
├── env.py           ← configuración del entorno de migración
├── script.py.mako   ← template para nuevas migraciones
└── versions/        ← archivos de migración generados
alembic.ini          ← configuración principal
```

---

## Configurar `alembic.ini`

```ini
# alembic.ini
[alembic]
# URL de conexión — usar variable de entorno (no hardcodear)
# La sobreescribimos en env.py
sqlalchemy.url = postgresql://user:pass@localhost/db
```

---

## Configurar `env.py` — la parte más importante

```python
# alembic/env.py
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import create_async_engine
from alembic import context

# Importar Base y TODOS los modelos para que Alembic los detecte
from src.mi_proyecto.database import Base
from src.mi_proyecto.config import settings

# Importar explícitamente los modelos para que sean detectados
from src.mi_proyecto.domain.models import (  # noqa
    User, Product, Category, Tag
)

config = context.config
if config.config_file_name:
    fileConfig(config.config_file_name)

# Sobreescribir la URL con la variable de entorno
config.set_main_option(
    "sqlalchemy.url",
    settings.database_url.replace("postgresql+asyncpg", "postgresql")  # sync para autogenerate
)

target_metadata = Base.metadata  # ← donde Alembic busca los modelos

def run_migrations_offline() -> None:
    """Genera SQL sin conectarse a la DB."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,   # detectar cambios de tipo
        compare_server_default=True,
    )
    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection):
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
    )
    with context.begin_transaction():
        context.run_migrations()

async def run_migrations_online() -> None:
    """Conecta a la DB y ejecuta las migraciones."""
    connectable = create_async_engine(settings.database_url)

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()

if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

---

## Flujo de trabajo

```bash
# 1. Crear migración automática (detecta cambios en los modelos)
uv run alembic revision --autogenerate -m "crear tabla products"
# Crea: alembic/versions/abc123def456_crear_tabla_products.py

# 2. Revisar el archivo generado SIEMPRE antes de aplicar
# (Alembic no detecta todo — renombrar columnas, por ejemplo)

# 3. Aplicar todas las migraciones pendientes
uv run alembic upgrade head

# 4. Aplicar solo la siguiente migración
uv run alembic upgrade +1

# 5. Ver estado actual
uv run alembic current

# 6. Ver historial
uv run alembic history --verbose

# 7. Hacer downgrade (revertir)
uv run alembic downgrade -1    # un paso atrás
uv run alembic downgrade base  # revertir todo (cuidado en producción)

# 8. Generar solo SQL (sin ejecutar)
uv run alembic upgrade head --sql > migration.sql
```

---

## Anatomía de una migración

```python
# alembic/versions/abc123def456_crear_tabla_products.py
"""crear tabla products

Revision ID: abc123def456
Revises: None  (o el ID anterior)
Create Date: 2025-01-15 10:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

# Identificadores de esta migración
revision = "abc123def456"
down_revision = None          # None = primera migración
branch_labels = None
depends_on = None

def upgrade() -> None:
    """Aplicar la migración (forward)."""
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(length=100), nullable=False),
        sa.Column("description", sa.String(length=500), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("name"),
    )
    op.create_index("ix_categories_name", "categories", ["name"])

    op.create_table(
        "products",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(length=200), nullable=False),
        sa.Column("price", sa.Numeric(precision=12, scale=2), nullable=False),
        sa.Column("stock", sa.Integer(), nullable=False, server_default="0"),
        sa.Column("category_id", sa.Integer(), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False, server_default="true"),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.ForeignKeyConstraint(["category_id"], ["categories.id"]),
        sa.PrimaryKeyConstraint("id"),
    )

def downgrade() -> None:
    """Revertir la migración (backward)."""
    op.drop_table("products")
    op.drop_table("categories")
```

---

## Operaciones comunes de migración

```python
# Agregar columna
def upgrade():
    op.add_column(
        "products",
        sa.Column("sku", sa.String(50), nullable=True)
    )
    # Poblar la columna antes de hacerla NOT NULL
    op.execute("UPDATE products SET sku = 'LEGACY-' || id::text WHERE sku IS NULL")
    op.alter_column("products", "sku", nullable=False)
    op.create_unique_constraint("uq_products_sku", "products", ["sku"])

def downgrade():
    op.drop_constraint("uq_products_sku", "products", type_="unique")
    op.drop_column("products", "sku")

# Renombrar columna (Alembic NO detecta esto — requiere migración manual)
def upgrade():
    op.alter_column("products", "name", new_column_name="title")

def downgrade():
    op.alter_column("products", "title", new_column_name="name")

# Agregar índice
def upgrade():
    op.create_index("ix_products_category_id", "products", ["category_id"])
    # Índice compuesto
    op.create_index("ix_products_category_price", "products", ["category_id", "price"])
    # Índice parcial (PostgreSQL)
    op.create_index(
        "ix_products_active",
        "products",
        ["name"],
        postgresql_where=sa.text("is_active = true")
    )

def downgrade():
    op.drop_index("ix_products_category_id", table_name="products")

# Ejecutar SQL directo
def upgrade():
    op.execute("""
        CREATE OR REPLACE VIEW product_summary AS
        SELECT p.id, p.name, c.name AS category, p.price
        FROM products p
        JOIN categories c ON c.id = p.category_id
        WHERE p.is_active = true
    """)
```

---

## Convenciones de nomenclatura

```python
# database.py — importante para que Alembic genere nombres predecibles
from sqlalchemy import MetaData

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

---

## Estrategia para producción

```bash
# NO correr alembic upgrade head automáticamente al iniciar la app
# en entornos serios — hacerlo como paso separado del deploy

# Opción A: en el pipeline de CI/CD antes del deploy
alembic upgrade head

# Opción B: en el entrypoint de Docker (para desarrollo/staging)
# entrypoint.sh
#!/bin/bash
set -e
echo "Aplicando migraciones..."
uv run alembic upgrade head
echo "Iniciando servidor..."
exec uv run uvicorn src.mi_proyecto.main:app --host 0.0.0.0 --port 8000
```

---

## Datos iniciales (seed) con Alembic

```python
# alembic/versions/...seed_categorias.py
def upgrade():
    op.bulk_insert(
        sa.table(
            "categories",
            sa.column("id", sa.Integer),
            sa.column("name", sa.String),
        ),
        [
            {"id": 1, "name": "Electrónica"},
            {"id": 2, "name": "Ropa"},
            {"id": 3, "name": "Alimentos"},
        ]
    )

def downgrade():
    op.execute("DELETE FROM categories WHERE id IN (1, 2, 3)")
```

---

## Checklist de migraciones

```
Antes de aplicar en producción:
[ ] Revisar el auto-generated — Alembic puede perder renombrados
[ ] Probar upgrade y downgrade en entorno de staging
[ ] Verificar que downgrade revierte todo correctamente
[ ] Si la tabla es grande: considerar índices CONCURRENTLY (PostgreSQL)
[ ] Hacer backup de la DB antes de migrar
[ ] Tener rollback plan documentado

Alembic NO detecta automáticamente:
[ ] Renombrar columnas (lo detecta como drop + add)
[ ] Renombrar tablas (ídem)
[ ] Cambios en funciones/vistas/triggers
[ ] Cambios en defaults complejos
```
