# FastAPI — Fundamentos

FastAPI es el framework web moderno de Python para APIs. Usa type hints de Python para generar documentación automática, validar input y serializar respuestas. Comparable a Gin en Go pero más declarativo.

---

## Instalación y setup

```toml
# pyproject.toml
[project]
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",  # servidor ASGI
    "pydantic>=2.9.0",
    "pydantic-settings>=2.6.0",
]
```

```bash
uv add fastapi uvicorn[standard] pydantic pydantic-settings
uv run uvicorn src.mi_proyecto.main:app --reload --port 8000
```

---

## Estructura de proyecto (Clean Architecture)

```
src/mi_proyecto/
├── main.py                  ← entry point, crea FastAPI app
├── config.py                ← settings con pydantic-settings
├── domain/
│   ├── models.py            ← entidades de dominio
│   └── exceptions.py        ← errores de dominio
├── repository/
│   └── product_repo.py      ← acceso a DB
├── usecase/
│   └── product_usecase.py   ← lógica de negocio
└── api/
    ├── deps.py              ← dependencias (get_session, get_current_user)
    ├── exception_handlers.py
    └── routers/
        ├── products.py
        └── auth.py
```

---

## App principal

```python
# src/mi_proyecto/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .api import products, auth
from .api.exception_handlers import app_error_handler
from .domain.exceptions import AppError
from .config import settings
from .database import engine, Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Se ejecuta al iniciar y cerrar la aplicación."""
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown
    await engine.dispose()

app = FastAPI(
    title="Mi API",
    version="1.0.0",
    lifespan=lifespan,
)

# CORS — permitir requests desde el frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Registrar handler de errores de dominio
app.add_exception_handler(AppError, app_error_handler)

# Registrar routers
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(products.router, prefix="/api/v1/products", tags=["products"])

@app.get("/health")
def health_check():
    return {"status": "ok"}
```

---

## Routers y path operations

```python
# api/routers/products.py
from fastapi import APIRouter, Depends, Query, Path, status
from sqlalchemy.ext.asyncio import AsyncSession

from ..deps import get_session, get_current_user
from ..schemas import ProductResponse, ProductCreate, ProductUpdate, ProductList
from ...usecase.product_usecase import ProductUsecase
from ...domain.models import User

router = APIRouter()

# GET /products?page=1&limit=20&category=electronics
@router.get("/", response_model=ProductList)
async def list_products(
    page: int = Query(default=1, ge=1, description="Número de página"),
    limit: int = Query(default=20, ge=1, le=100, description="Items por página"),
    category: str | None = Query(default=None, description="Filtrar por categoría"),
    session: AsyncSession = Depends(get_session),
):
    usecase = ProductUsecase(session)
    return await usecase.list_products(page=page, limit=limit, category=category)

# GET /products/123
@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: int = Path(gt=0, description="ID del producto"),
    session: AsyncSession = Depends(get_session),
):
    usecase = ProductUsecase(session)
    return await usecase.get_product(product_id)

# POST /products
@router.post("/", response_model=ProductResponse, status_code=status.HTTP_201_CREATED)
async def create_product(
    data: ProductCreate,  # request body — validado con Pydantic
    session: AsyncSession = Depends(get_session),
    current_user: User = Depends(get_current_user),  # requiere autenticación
):
    usecase = ProductUsecase(session)
    return await usecase.create_product(data, created_by=current_user.id)

# PATCH /products/123 — actualización parcial
@router.patch("/{product_id}", response_model=ProductResponse)
async def update_product(
    product_id: int = Path(gt=0),
    data: ProductUpdate = ...,
    session: AsyncSession = Depends(get_session),
    current_user: User = Depends(get_current_user),
):
    usecase = ProductUsecase(session)
    return await usecase.update_product(product_id, data)

# DELETE /products/123
@router.delete("/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(
    product_id: int = Path(gt=0),
    session: AsyncSession = Depends(get_session),
    current_user: User = Depends(get_current_user),
):
    usecase = ProductUsecase(session)
    await usecase.delete_product(product_id)
```

---

## Dependency Injection con `Depends`

```python
# api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from ..database import AsyncSessionLocal
from ..domain.models import User
from ..auth import decode_access_token

# Generador — FastAPI abre/cierra la sesión automáticamente
async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> User:
    """Dependencia que verifica el token JWT y retorna el usuario."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Credenciales inválidas",
        headers={"WWW-Authenticate": "Bearer"},
    )

    payload = decode_access_token(token)
    if not payload:
        raise credentials_exception

    user = await session.get(User, payload.sub)
    if not user or not user.is_active:
        raise credentials_exception

    return user

# Dependencia de admin — reutiliza get_current_user
async def require_admin(current_user: User = Depends(get_current_user)) -> User:
    if not current_user.is_admin:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)
    return current_user
```

---

## Middleware

```python
# Middleware de logging con tiempo de respuesta
import time
from fastapi import Request

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    logger.info(
        "%s %s → %d (%.3fs)",
        request.method,
        request.url.path,
        response.status_code,
        duration
    )

    response.headers["X-Process-Time"] = str(duration)
    return response
```

---

## Background Tasks

```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str, name: str) -> None:
    """Se ejecuta después de retornar la respuesta."""
    await email_service.send(
        to=email,
        subject="Bienvenido",
        body=f"Hola {name}!"
    )

@router.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(
    data: UserCreate,
    background_tasks: BackgroundTasks,
    session: AsyncSession = Depends(get_session),
):
    user = await UserUsecase(session).create(data)
    # Se agrega al queue — el email se envía después de retornar 201
    background_tasks.add_task(send_welcome_email, user.email, user.name)
    return user
```

---

## Request y Response personalizados

```python
from fastapi import Request
from fastapi.responses import JSONResponse, StreamingResponse
import json

# Acceder al request crudo
@router.post("/webhook")
async def webhook(request: Request):
    body = await request.body()           # bytes
    data = await request.json()           # dict
    headers = dict(request.headers)      # headers
    client_ip = request.client.host      # IP cliente
    return {"received": True}

# Streaming response — para archivos grandes o eventos SSE
@router.get("/export/products")
async def export_products(session: AsyncSession = Depends(get_session)):
    async def generate():
        async for product in iter_all_products(session):
            yield json.dumps(product.model_dump()) + "\n"

    return StreamingResponse(
        generate(),
        media_type="application/x-ndjson",
        headers={"Content-Disposition": "attachment; filename=products.ndjson"}
    )
```

---

## Documentación automática

FastAPI genera automáticamente:

- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`
- **OpenAPI JSON**: `http://localhost:8000/openapi.json`

Para enriquecer la documentación:

```python
@router.get(
    "/{product_id}",
    response_model=ProductResponse,
    summary="Obtener producto por ID",
    description="Retorna los detalles completos de un producto.",
    response_description="Producto encontrado",
    responses={
        404: {"description": "Producto no encontrado"},
        401: {"description": "No autenticado"},
    },
)
async def get_product(product_id: int):
    ...
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Lógica de negocio en el router — viola Clean Architecture
@router.get("/products/{id}")
async def get_product(id: int, session: AsyncSession = Depends(get_session)):
    result = await session.execute(
        select(ProductModel).where(ProductModel.id == id)  # DB en router
    )
    product = result.scalar_one_or_none()
    if not product:
        raise HTTPException(404, "No encontrado")  # manejo de errores ad-hoc
    return product  # devuelve modelo de ORM directamente (expone internals)

# Sin response_model — cualquier cambio en el modelo ORM rompe la API
@router.get("/users/{id}")
async def get_user(id: int):
    return await db.get_user(id)  # puede devolver campos sensibles (password_hash)
```

### Profesional

```python
# Router solo coordina — delega al usecase
@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: int = Path(gt=0),
    session: AsyncSession = Depends(get_session),
):
    # Usecase maneja la lógica, lanza NotFoundError si no existe
    return await ProductUsecase(session).get_product(product_id)

# response_model garantiza qué campos se devuelven
# El handler global convierte NotFoundError → 404 automáticamente
```
