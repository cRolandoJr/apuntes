# Funciones y Decoradores

Los decoradores son uno de los conceptos más poderosos y distintivos de Python. Los vas a encontrar en todos los frameworks (FastAPI, Django, pytest, etc.).

---

## Funciones — características clave

```python
# Argumentos posicionales y keyword
def create_product(name: str, price: float, stock: int = 0) -> dict:
    return {"name": name, "price": price, "stock": stock}

create_product("Laptop", 1500.0)           # posicional
create_product("Laptop", price=1500.0)     # keyword
create_product(name="Laptop", price=1500.0, stock=10)  # todo keyword

# *args — argumentos posicionales variables (tupla)
def sum_all(*numbers: int) -> int:
    return sum(numbers)

sum_all(1, 2, 3, 4, 5)  # → 15

# **kwargs — argumentos keyword variables (dict)
def log_event(event: str, **metadata) -> None:
    print(f"[{event}] {metadata}")

log_event("USER_LOGIN", user_id="123", ip="192.168.1.1")

# Positional-only (/) y keyword-only (*)
def create(name: str, /, *, price: float) -> dict:
    # name: solo posicional (no se puede llamar como keyword)
    # price: solo keyword (no se puede llamar como posicional)
    return {"name": name, "price": price}

create("Laptop", price=1500.0)  # OK
# create(name="Laptop", price=1500.0)  # TypeError: name es positional-only
```

---

## Funciones como objetos de primera clase

En Python, las funciones son objetos — podés pasarlas como argumentos, retornarlas, guardarlas en variables.

```python
from typing import Callable

# Función que recibe otra función como argumento
def apply(func: Callable[[int], int], value: int) -> int:
    return func(value)

apply(lambda x: x * 2, 5)  # → 10

# Función que retorna otra función (Higher-Order Function)
def multiplier(factor: int) -> Callable[[int], int]:
    def multiply(x: int) -> int:
        return x * factor
    return multiply

double = multiplier(2)
triple = multiplier(3)
double(5)  # → 10
triple(5)  # → 15

# Lambda — función anónima de una expresión
numbers = [3, 1, 4, 1, 5, 9]
sorted(numbers, key=lambda x: -x)  # ordenar descendente

products = [{"name": "B", "price": 200}, {"name": "A", "price": 100}]
sorted(products, key=lambda p: p["price"])
```

---

## Closures

```python
def make_validator(min_val: float, max_val: float) -> Callable[[float], bool]:
    """La función interna 'captura' min_val y max_val del scope exterior."""
    def validate(value: float) -> bool:
        return min_val <= value <= max_val
    return validate

is_valid_price = make_validator(0.01, 999_999.99)
is_valid_percentage = make_validator(0.0, 100.0)

is_valid_price(1500.0)       # True
is_valid_price(-10.0)        # False
is_valid_percentage(50.0)    # True
```

---

## Decoradores — qué son y cómo funcionan

Un decorador es una función que recibe una función y retorna una función nueva (con comportamiento adicional).

```python
# Sin decorador — lo que el decorador hace internamente
def mi_funcion():
    print("hola")

def logger(func):
    def wrapper(*args, **kwargs):
        print(f"Llamando a {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} terminó")
        return result
    return wrapper

mi_funcion = logger(mi_funcion)  # decoración manual

# Con sintaxis de decorador — equivalente exacto
@logger
def mi_funcion():
    print("hola")
```

### Decorador con argumentos — tres niveles de functions

```python
import functools
import time

def retry(max_attempts: int = 3, delay: float = 1.0):
    """Decorador con parámetros — requiere 3 niveles."""
    def decorator(func):
        @functools.wraps(func)  # preserva __name__, __doc__, etc. del original
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    print(f"Intento {attempt} fallido: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise last_error
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def fetch_data(url: str) -> dict:
    # puede fallar por red
    ...
```

### Decorador que preserva tipo (type-safe con ParamSpec)

```python
from typing import TypeVar, ParamSpec, Callable
import functools

P = ParamSpec("P")
T = TypeVar("T")

def timer(func: Callable[P, T]) -> Callable[P, T]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} tardó {elapsed:.3f}s")
        return result
    return wrapper
```

---

## Decoradores de clase — property, classmethod, staticmethod

```python
class ProductService:
    _instance = None

    def __init__(self, db_url: str):
        self._db_url = db_url
        self._connection_count = 0

    # property — acceso como atributo, lógica de getter
    @property
    def connection_count(self) -> int:
        return self._connection_count

    # setter — validación al asignar
    @connection_count.setter
    def connection_count(self, value: int) -> None:
        if value < 0:
            raise ValueError("no puede ser negativo")
        self._connection_count = value

    # classmethod — recibe la clase, no la instancia. Factory pattern.
    @classmethod
    def from_env(cls) -> "ProductService":
        import os
        return cls(db_url=os.environ["DATABASE_URL"])

    # staticmethod — no recibe cls ni self. Utilidad relacionada.
    @staticmethod
    def validate_price(price: float) -> bool:
        return price > 0

# Uso
svc = ProductService.from_env()  # factory
ProductService.validate_price(100.0)  # sin instancia
svc.connection_count = 5  # usa el setter
print(svc.connection_count)  # usa el getter (sin llamar como función)
```

---

## Decoradores en el mundo real

Los frameworks usan decoradores para todo:

```python
# FastAPI — registrar rutas
@app.get("/products/{product_id}")
async def get_product(product_id: str) -> Product:
    ...

# FastAPI — dependencias
@app.post("/products")
async def create_product(
    product: ProductCreate,
    current_user: User = Depends(get_current_user),  # inyección
    db: AsyncSession = Depends(get_db),
):
    ...

# pytest — fixtures
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session
    session.rollback()

# dataclasses
@dataclass(frozen=True)
class Point:
    x: float
    y: float

# cache — memoización
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# cache sin límite (Python 3.9+)
from functools import cache

@cache
def get_config() -> dict:
    return load_from_file()  # se llama una sola vez
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Decorador sin functools.wraps — rompe debugging e introspección
def log(func):
    def wrapper(*args, **kwargs):
        print("llamando")
        return func(*args, **kwargs)
    return wrapper  # wrapper.__name__ = "wrapper", no "mi_funcion"

# Lógica en cada handler en lugar de en decorador
def get_product(id):
    if not is_authenticated():
        raise Exception("no auth")
    # ... misma verificación en 20 endpoints
```

### Profesional

```python
# functools.wraps preserva metadata
def require_auth(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        if not is_authenticated():
            raise UnauthorizedError()
        return func(*args, **kwargs)
    return wrapper

# Un decorador, aplicado a todos los endpoints que lo necesiten
@app.get("/admin/products")
@require_auth
async def list_admin_products():
    ...
```
