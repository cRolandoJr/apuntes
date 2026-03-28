# Testing con pytest

pytest es el framework de testing estándar en Python. Más poderoso que el `unittest` de la stdlib. En proyectos FastAPI + SQLAlchemy el testing async requiere algunas configuraciones extra.

---

## Instalación

```bash
uv add --dev pytest pytest-asyncio pytest-cov httpx
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"      # todas las coroutines de test son async automáticamente
addopts = [
    "--cov=src",           # coverage de src/
    "--cov-report=term-missing",
    "--cov-fail-under=80",  # falla si coverage < 80%
]
```

---

## Estructura de tests

```
tests/
├── conftest.py           ← fixtures compartidos
├── unit/
│   ├── test_product_usecase.py
│   └── test_validators.py
├── integration/
│   ├── test_product_repo.py
│   └── test_auth.py
└── e2e/
    └── test_api_products.py
```

---

## Tests básicos — funciones simples

```python
# tests/unit/test_validators.py
from mi_proyecto.auth.password import hash_password, verify_password
from mi_proyecto.domain.exceptions import ValidationError

def test_password_hash_is_not_plain():
    hashed = hash_password("mysecretpassword")
    assert hashed != "mysecretpassword"
    assert len(hashed) > 20

def test_password_verify_correct():
    plain = "mysecretpassword123"
    hashed = hash_password(plain)
    assert verify_password(plain, hashed) is True

def test_password_verify_wrong():
    hashed = hash_password("correct")
    assert verify_password("wrong", hashed) is False

# pytest.raises — verificar que lanza excepción
def test_validation_error_has_code():
    with pytest.raises(ValidationError) as exc_info:
        raise ValidationError("campo requerido", field="name")

    assert exc_info.value.code == "INVALID_INPUT"
    assert exc_info.value.field == "name"
```

---

## Fixtures — `conftest.py`

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from httpx import AsyncClient, ASGITransport

from mi_proyecto.main import app
from mi_proyecto.database import Base
from mi_proyecto.api.deps import get_session

# Base de datos en memoria para tests (SQLite async)
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest_asyncio.fixture(scope="session")
async def engine():
    """Engine compartido para toda la sesión de tests."""
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest_asyncio.fixture
async def session(engine):
    """Sesión de DB que se revierte después de cada test."""
    async with engine.begin() as conn:
        # Usar savepoint para rollback automático al final del test
        async with AsyncSession(bind=conn) as session:
            await session.begin_nested()
            yield session
            await session.rollback()

@pytest_asyncio.fixture
async def client(session):
    """Cliente HTTP que usa la sesión de test."""
    # Sobreescribir la dependencia de sesión con la de test
    app.dependency_overrides[get_session] = lambda: session

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()

@pytest_asyncio.fixture
async def product(session):
    """Fixture: un producto de prueba en la DB."""
    from mi_proyecto.domain.models import Product, Category

    category = Category(name="Electrónica")
    session.add(category)
    await session.flush()

    product = Product(
        name="Laptop de prueba",
        price=1500,
        stock=10,
        sku="TEST-001",
        category_id=category.id,
    )
    session.add(product)
    await session.flush()
    return product

@pytest.fixture
def auth_headers(user_token):
    """Headers de autenticación para el cliente."""
    return {"Authorization": f"Bearer {user_token}"}
```

---

## Tests de API (e2e)

```python
# tests/e2e/test_api_products.py
import pytest
from httpx import AsyncClient

async def test_list_products_empty(client: AsyncClient):
    response = await client.get("/api/v1/products/")
    assert response.status_code == 200
    data = response.json()
    assert data["items"] == []
    assert data["total"] == 0

async def test_get_product_found(client: AsyncClient, product):
    response = await client.get(f"/api/v1/products/{product.id}")
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == product.name
    assert data["id"] == product.id

async def test_get_product_not_found(client: AsyncClient):
    response = await client.get("/api/v1/products/9999")
    assert response.status_code == 404
    assert response.json()["error"] == "NOT_FOUND"

async def test_create_product_requires_auth(client: AsyncClient):
    response = await client.post(
        "/api/v1/products/",
        json={"name": "Test", "price": 100, "sku": "TEST-002", "category_id": 1}
    )
    assert response.status_code == 401

async def test_create_product_success(client: AsyncClient, auth_headers):
    payload = {
        "name": "Nuevo producto",
        "price": "299.99",
        "sku": "PROD-003",
        "category_id": 1,
        "stock": 50,
    }
    response = await client.post(
        "/api/v1/products/",
        json=payload,
        headers=auth_headers,
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == payload["name"]
    assert data["id"] is not None

async def test_create_product_invalid_price(client: AsyncClient, auth_headers):
    response = await client.post(
        "/api/v1/products/",
        json={"name": "Test", "price": -10, "sku": "TST-001", "category_id": 1},
        headers=auth_headers,
    )
    assert response.status_code == 422  # Pydantic validation error
```

---

## Tests unitarios con mocks

```python
# tests/unit/test_product_usecase.py
from unittest.mock import AsyncMock, MagicMock, patch
import pytest

from mi_proyecto.usecase.product_usecase import ProductUsecase
from mi_proyecto.domain.exceptions import NotFoundError, ConflictError

@pytest.fixture
def mock_repo():
    repo = AsyncMock()
    repo.get_by_id = AsyncMock()
    repo.create = AsyncMock()
    repo.exists_by_name = AsyncMock(return_value=False)
    return repo

async def test_get_product_raises_not_found(mock_repo):
    mock_repo.get_by_id.return_value = None
    usecase = ProductUsecase.__new__(ProductUsecase)
    usecase._repo = mock_repo

    with pytest.raises(NotFoundError):
        await usecase.get_product(9999)

    mock_repo.get_by_id.assert_awaited_once_with(9999)

async def test_create_product_raises_conflict_on_duplicate_name(mock_repo):
    mock_repo.exists_by_name.return_value = True  # nombre ya existe
    usecase = ProductUsecase.__new__(ProductUsecase)
    usecase._repo = mock_repo

    with pytest.raises(ConflictError) as exc_info:
        await usecase.create_product(MagicMock(name="Laptop"))

    assert "Laptop" in str(exc_info.value)
```

---

## `@pytest.mark.parametrize` — tests con múltiples casos

```python
import pytest

@pytest.mark.parametrize("price,expected_valid", [
    (100, True),
    (0.01, True),
    (0, False),        # precio 0 no válido
    (-10, False),      # precio negativo
    (None, False),     # None no válido
])
def test_product_price_validation(price, expected_valid):
    from pydantic import ValidationError
    from mi_proyecto.api.schemas import ProductCreate

    data = {"name": "Test", "sku": "TST-001", "category_id": 1}
    if price is not None:
        data["price"] = price

    if expected_valid:
        product = ProductCreate(**data)  # no lanza
        assert product.price == price
    else:
        with pytest.raises(ValidationError):
            ProductCreate(**data)

@pytest.mark.parametrize("email", [
    "valid@example.com",
    "user.name+tag@domain.co",
    "user@subdomain.example.com",
])
def test_valid_emails(email):
    from pydantic import EmailStr
    # ...

@pytest.mark.parametrize("email", [
    "not-an-email",
    "@domain.com",
    "user@",
])
def test_invalid_emails(email):
    from pydantic import ValidationError
    # ...
```

---

## Coverage y ejecutar tests

```bash
# Ejecutar todos los tests
uv run pytest

# Verbose
uv run pytest -v

# Solo un archivo
uv run pytest tests/e2e/test_api_products.py

# Solo tests que coinciden con un patrón
uv run pytest -k "test_create"

# Con reporte de coverage en HTML
uv run pytest --cov=src --cov-report=html
# abre htmlcov/index.html

# Detener al primer fallo
uv run pytest -x

# Mostrar print() output (por defecto se captura)
uv run pytest -s

# Ejecutar tests marcados
uv run pytest -m "not slow"      # excluir tests lentos
```

```python
# Marcar tests
@pytest.mark.slow    # uv run pytest -m "not slow" los salta
async def test_heavy_operation():
    ...

@pytest.mark.skip(reason="WIP")
def test_feature_not_ready():
    ...

@pytest.mark.xfail(reason="Bug conocido #123")
def test_known_bug():
    ...
```
