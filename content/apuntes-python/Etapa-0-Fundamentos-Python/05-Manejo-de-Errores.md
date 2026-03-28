# Manejo de Errores en Python

Python usa excepciones para todo — diferente a Go donde los errores son valores. Entender cuándo lanzar, cuándo capturar, y cómo crear una jerarquía de errores de dominio es esencial.

---

## La jerarquía built-in

```
BaseException
├── SystemExit           ← sys.exit()
├── KeyboardInterrupt    ← Ctrl+C
├── GeneratorExit        ← generadores
└── Exception            ← ← ← la mayoría de errores de tu código
    ├── ValueError           ← valor incorrecto para el tipo
    ├── TypeError            ← tipo incorrecto
    ├── KeyError             ← clave no existe en dict
    ├── IndexError           ← índice fuera de rango
    ├── AttributeError       ← atributo no existe
    ├── FileNotFoundError    ← archivo no existe (IOError)
    ├── PermissionError      ← sin permisos
    ├── ImportError          ← no se puede importar
    ├── OSError              ← error del SO
    ├── RuntimeError         ← error en runtime
    └── NotImplementedError  ← método abstracto no implementado
```

**Regla**: capturá `Exception` solo cuando realmente querés capturar todo. Capturá el tipo más específico posible.

---

## try/except/else/finally

```python
def read_config(path: str) -> dict:
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        # archivo no existe
        return {}
    except json.JSONDecodeError as e:
        # archivo existe pero no es JSON válido
        raise ValueError(f"Config inválida en {path}: {e}") from e
    except PermissionError as e:
        raise PermissionError(f"Sin permisos para leer {path}") from e
    else:
        # Se ejecuta si NO hubo excepción en try
        # Útil para código que solo debe ejecutarse si todo fue bien
        print("Config cargada correctamente")
    finally:
        # Se ejecuta SIEMPRE (con o sin excepción)
        # Para cleanup: cerrar conexiones, archivos, etc.
        print("Fin del intento de carga")
```

### Capturar múltiples tipos

```python
try:
    data = process(input_data)
except (ValueError, TypeError) as e:
    # manejar ambos igual
    logger.error("datos inválidos: %s", e)
except KeyError as e:
    logger.error("clave faltante: %s", e)
except Exception as e:
    # catch-all — loguear y re-lanzar
    logger.exception("error inesperado")
    raise  # re-lanza la misma excepción con su traceback original
```

---

## Excepciones de dominio — jerarquía propia

```python
# domain/exceptions.py

class AppError(Exception):
    """Clase base de todos los errores de dominio."""

    def __init__(self, message: str, code: str = "INTERNAL_ERROR") -> None:
        super().__init__(message)
        self.message = message
        self.code = code

    def __str__(self) -> str:
        return f"[{self.code}] {self.message}"

class NotFoundError(AppError):
    def __init__(self, message: str = "Recurso no encontrado") -> None:
        super().__init__(message, code="NOT_FOUND")

class ValidationError(AppError):
    def __init__(self, message: str, field: str | None = None) -> None:
        super().__init__(message, code="INVALID_INPUT")
        self.field = field

class UnauthorizedError(AppError):
    def __init__(self, message: str = "No autenticado") -> None:
        super().__init__(message, code="UNAUTHORIZED")

class ForbiddenError(AppError):
    def __init__(self, message: str = "Sin permisos") -> None:
        super().__init__(message, code="FORBIDDEN")

class ConflictError(AppError):
    def __init__(self, message: str) -> None:
        super().__init__(message, code="CONFLICT")

# Uso en el usecase
def create_product(self, data: CreateProductInput) -> Product:
    if not data.name:
        raise ValidationError("El nombre es requerido", field="name")

    if self.repo.exists_by_name(data.name):
        raise ConflictError(f"Ya existe un producto con el nombre '{data.name}'")

    return self.repo.create(data)
```

### Mapear errores de dominio a HTTP en FastAPI

```python
# api/exception_handlers.py
from fastapi import Request
from fastapi.responses import JSONResponse
from mi_proyecto.domain.exceptions import (
    AppError, NotFoundError, ValidationError,
    UnauthorizedError, ForbiddenError, ConflictError
)

ERROR_TO_STATUS = {
    NotFoundError: 404,
    ValidationError: 422,
    UnauthorizedError: 401,
    ForbiddenError: 403,
    ConflictError: 409,
    AppError: 500,
}

async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    status_code = ERROR_TO_STATUS.get(type(exc), 500)
    return JSONResponse(
        status_code=status_code,
        content={"error": exc.code, "message": exc.message},
    )

# Registrar en la app
app.add_exception_handler(AppError, app_error_handler)
```

---

## Exception chaining — preservar contexto

```python
# raise X from Y — encadena la excepción original
try:
    result = db.query(Product, product_id)
except psycopg2.Error as e:
    # preserva el error de DB como contexto
    raise NotFoundError(f"Producto {product_id} no encontrado") from e
    # traceback mostrará AMBAS excepciones

# raise X from None — suprimir la excepción original
try:
    value = int(user_input)
except ValueError:
    raise ValidationError("Debe ser un número") from None
    # traceback solo muestra ValidationError (sin ruido de ValueError)
```

---

## Context managers para manejo de recursos

```python
# contextlib.contextmanager — crear context managers con generadores
from contextlib import contextmanager, asynccontextmanager
import psycopg2

@contextmanager
def get_db_connection(dsn: str):
    conn = psycopg2.connect(dsn)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()  # siempre se ejecuta

with get_db_connection(DSN) as conn:
    cursor = conn.cursor()
    cursor.execute("INSERT INTO products ...")
    # commit automático si no hubo excepción
    # rollback si la hubo

# Async context manager
@asynccontextmanager
async def get_async_session():
    async with AsyncSession(engine) as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

---

## Suprimir excepciones específicas

```python
from contextlib import suppress

# En lugar de:
try:
    os.remove("temp.txt")
except FileNotFoundError:
    pass

# Usar suppress:
with suppress(FileNotFoundError):
    os.remove("temp.txt")

# suppress múltiples
with suppress(FileNotFoundError, PermissionError):
    shutil.rmtree("/tmp/old_cache")
```

---

## Logging vs raising

```python
import logging

logger = logging.getLogger(__name__)

# REGLA: loguear O lanzar, nunca los dos (double-logging anti-pattern)

# MAL — se loguea en el repositorio Y en el handler
def get_product_bad(id: str) -> Product:
    try:
        return db.get(id)
    except Exception as e:
        logger.error("Error al obtener producto: %s", e)  # log aquí
        raise  # Y también se propaga

def handle_request_bad():
    try:
        product = get_product_bad("123")
    except Exception as e:
        logger.error("Error en el request: %s", e)  # log OTRA VEZ

# BIEN — sólo el nivel más alto loguea
def get_product(id: str) -> Product:
    try:
        return db.get(id)
    except RecordNotFound:
        raise NotFoundError(f"Producto {id} no encontrado")  # solo transforma, no loguea

def handle_request():
    try:
        product = get_product("123")
    except AppError as e:
        logger.error("Error manejado: %s", e.code)  # un solo log, en el boundary
        raise
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Capturar todo sin re-lanzar — tragar errores
try:
    result = process_data()
except:  # MAL: captura hasta KeyboardInterrupt
    print("algo salió mal")

# Excepciones genéricas sin información útil
raise Exception("error")  # ¿qué error? ¿dónde? ¿por qué?
```

### Profesional

```python
# Solo capturar lo que podés manejar
try:
    result = process_data()
except ValidationError as e:
    return ErrorResponse(code=e.code, message=e.message)
# Los demás errores se propagan — el handler global los captura

# Excepciones de dominio con información completa
raise ValidationError("precio inválido: debe ser mayor a 0", field="price")

# Exception chaining para no perder contexto
try:
    row = db.execute(query)
except DBError as e:
    raise NotFoundError(f"Producto {id} no encontrado") from e
```
