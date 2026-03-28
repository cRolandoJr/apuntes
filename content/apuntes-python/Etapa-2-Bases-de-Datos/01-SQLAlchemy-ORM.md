# SQLAlchemy 2.0 — ORM Asíncrono

SQLAlchemy 2.0 (2023+) cambió el API completamente: ya no hay `session.query()`, ahora se usa `select()` explícito al estilo SQL. La versión async usa `asyncpg` como driver.

---

## Instalación

```bash
uv add sqlalchemy[asyncio] asyncpg greenlet
```

---

## Configuración de la conexión

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from .config import settings

# Engine — pool de conexiones
engine = create_async_engine(
    settings.database_url,      # postgresql+asyncpg://user:pass@host/db
    echo=settings.debug,        # loguear SQL (solo en desarrollo)
    pool_size=10,               # conexiones permanentes en el pool
    max_overflow=20,            # conexiones extra en pico
    pool_timeout=30,            # segundos esperando conexión disponible
    pool_recycle=1800,          # reciclar conexiones cada 30 min
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,     # evita lazy loading después del commit
)

class Base(DeclarativeBase):
    pass

# .env
# DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/mi_db
```

---

## Modelos (tablas)

```python
# domain/models.py
from datetime import datetime, timezone
from decimal import Decimal
from sqlalchemy import String, Numeric, ForeignKey, Boolean, func
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID as PGUUID
import uuid

from ..database import Base

class TimestampMixin:
    """Mixin con created_at y updated_at automáticos."""
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc),
        server_default=func.now(),
        onupdate=lambda: datetime.now(timezone.utc),
    )

class Category(Base, TimestampMixin):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    description: Mapped[str | None] = mapped_column(String(500))

    # Relación: una categoría tiene muchos productos
    products: Mapped[list["Product"]] = relationship(
        back_populates="category",
        lazy="select",     # lazy loading por defecto
    )

class Product(Base, TimestampMixin):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200), index=True)
    description: Mapped[str | None] = mapped_column(String(5000))
    price: Mapped[Decimal] = mapped_column(Numeric(precision=12, scale=2))
    stock: Mapped[int] = mapped_column(default=0)
    sku: Mapped[str] = mapped_column(String(50), unique=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    # FK
    category_id: Mapped[int] = mapped_column(ForeignKey("categories.id"))
    category: Mapped["Category"] = relationship(back_populates="products")

    # Muchos a muchos con tags
    tags: Mapped[list["Tag"]] = relationship(
        secondary="product_tags",
        back_populates="products",
    )

class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    products: Mapped[list["Product"]] = relationship(
        secondary="product_tags", back_populates="tags"
    )

# Tabla de asociación product_tags (many-to-many)
from sqlalchemy import Table, Column
product_tags = Table(
    "product_tags",
    Base.metadata,
    Column("product_id", ForeignKey("products.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)
```

---

## Repository pattern — CRUD asíncrono

```python
# repository/product_repo.py
from sqlalchemy import select, update, delete, func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from ..domain.models import Product
from ..api.schemas import ProductCreate, ProductUpdate

class ProductRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_by_id(self, product_id: int) -> Product | None:
        return await self._session.get(Product, product_id)

    async def get_by_sku(self, sku: str) -> Product | None:
        result = await self._session.execute(
            select(Product).where(Product.sku == sku)
        )
        return result.scalar_one_or_none()

    async def list(
        self,
        *,
        page: int = 1,
        limit: int = 20,
        category_id: int | None = None,
        is_active: bool = True,
    ) -> tuple[list[Product], int]:
        """Retorna (productos, total)."""
        query = (
            select(Product)
            .where(Product.is_active == is_active)
            .options(selectinload(Product.category))  # eager load categoría
        )

        if category_id:
            query = query.where(Product.category_id == category_id)

        # Total (sin paginación)
        count_query = select(func.count()).select_from(query.subquery())
        total = await self._session.scalar(count_query)

        # Con paginación
        offset = (page - 1) * limit
        result = await self._session.execute(
            query.offset(offset).limit(limit).order_by(Product.name)
        )
        products = list(result.scalars().all())

        return products, total or 0

    async def create(self, data: ProductCreate) -> Product:
        product = Product(**data.model_dump())
        self._session.add(product)
        await self._session.flush()   # obtiene el ID sin hacer commit
        await self._session.refresh(product)
        return product

    async def update(self, product_id: int, data: ProductUpdate) -> Product | None:
        product = await self.get_by_id(product_id)
        if not product:
            return None

        # Actualizar solo los campos enviados
        update_data = data.model_dump(exclude_none=True)
        for field, value in update_data.items():
            setattr(product, field, value)

        await self._session.flush()
        await self._session.refresh(product)
        return product

    async def delete(self, product_id: int) -> bool:
        product = await self.get_by_id(product_id)
        if not product:
            return False
        await self._session.delete(product)
        return True

    async def exists_by_name(self, name: str) -> bool:
        result = await self._session.execute(
            select(Product.id).where(Product.name == name).limit(1)
        )
        return result.scalar_one_or_none() is not None
```

---

## Transacciones

```python
# La sesión maneja la transacción automáticamente a través del context manager
# pero el commit/rollback debe ser explícito

# Patrón 1: Unit of Work en el usecase
class ProductUsecase:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._repo = ProductRepository(session)

    async def create_product_with_tags(
        self, data: ProductCreate, tag_names: list[str]
    ) -> Product:
        # Todo dentro de la misma sesión = misma transacción
        product = await self._repo.create(data)

        for tag_name in tag_names:
            tag = Tag(name=tag_name)
            self._session.add(tag)
            product.tags.append(tag)

        # flush envía a DB sin commit (mantiene la transacción abierta)
        await self._session.flush()
        await self._session.refresh(product)
        return product
        # El commit lo hace el caller (el router o el dep de sesión)

# Patrón 2: Savepoint para transacciones anidadas
async def complex_operation(session: AsyncSession):
    async with session.begin_nested() as nested:  # SAVEPOINT
        try:
            await do_something_risky(session)
            # nested.commit() — libera el savepoint
        except SpecificError:
            await nested.rollback()  # solo deshace hasta el savepoint
            # La transacción principal sigue activa

    # Continuar con la transacción principal...
    await session.commit()
```

---

## Eager loading — evitar N+1

```python
from sqlalchemy.orm import selectinload, joinedload

# PROBLEMA N+1: cargar 100 productos, luego 1 query por categoría = 101 queries
products_bad = (await session.execute(select(Product))).scalars().all()
for p in products_bad:
    print(p.category.name)  # MAL — cada acceso hace una query (lazy loading)

# SOLUCIÓN 1: selectinload — hace 2 queries (uno para products, uno para categories)
# Recomendado para colecciones (1-to-many, many-to-many)
result = await session.execute(
    select(Product)
    .options(
        selectinload(Product.category),          # carga categorías
        selectinload(Product.tags),              # carga tags
    )
    .where(Product.is_active == True)
)
products = result.scalars().all()

# SOLUCIÓN 2: joinedload — hace 1 query con JOIN
# Recomendado para relaciones singulares (many-to-one)
result = await session.execute(
    select(Product)
    .options(joinedload(Product.category))
    .where(Product.id == product_id)
)
product = result.unique().scalar_one_or_none()
```

---

## Queries avanzadas

```python
from sqlalchemy import and_, or_, not_, between, ilike, case

# AND / OR
query = select(Product).where(
    and_(
        Product.is_active == True,
        or_(
            Product.price < 100,
            Product.category_id == 5,
        )
    )
)

# LIKE case-insensitive
query = select(Product).where(
    Product.name.ilike(f"%{search_term}%")
)

# BETWEEN
query = select(Product).where(
    between(Product.price, min_price, max_price)
)

# ORDER BY múltiple
query = select(Product).order_by(
    Product.category_id.asc(),
    Product.price.desc(),
)

# Aggregate
from sqlalchemy import func
result = await session.execute(
    select(
        Category.name,
        func.count(Product.id).label("product_count"),
        func.avg(Product.price).label("avg_price"),
    )
    .join(Product, Product.category_id == Category.id)
    .group_by(Category.id, Category.name)
    .having(func.count(Product.id) > 0)
)
stats = result.all()  # lista de Row namedtuples
for row in stats:
    print(row.name, row.product_count, row.avg_price)
```

---

## Session como dependencia en FastAPI

```python
# database.py
async def get_session():
    async with AsyncSessionLocal() as session:
        async with session.begin():  # auto-commit o rollback al salir
            yield session

# api/deps.py — simplemente reexportar o wrappear
from ..database import get_session as _get_session
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends
from typing import AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```
