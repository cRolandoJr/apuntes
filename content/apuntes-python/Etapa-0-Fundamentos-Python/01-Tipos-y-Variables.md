# Tipos y Variables en Python

Python es dinámicamente tipado, pero desde Python 3.5+ tiene **type hints** opcionales. En código profesional siempre se usan — mejoran el autocompletado, detectan errores en CI y documentan el código.

---

## Tipos primitivos

```python
# Enteros — sin límite de tamaño (arbitrariamente grandes)
x: int = 42
big: int = 10 ** 100  # Python lo maneja sin overflow

# Flotantes — doble precisión IEEE 754
price: float = 19.99
# Cuidado con imprecisiones:
0.1 + 0.2  # → 0.30000000000000004

# Para dinero siempre usar Decimal
from decimal import Decimal
price = Decimal("19.99")  # string, no float — evita imprecisión

# Strings — inmutables, Unicode por defecto
name: str = "Laptop Pro"
multiline: str = """
  Texto en
  varias líneas
"""

# f-strings — la forma moderna (Python 3.6+)
greeting = f"Hola, {name}. Precio: ${price:.2f}"

# Booleanos
is_active: bool = True
is_deleted: bool = False

# None — equivalente a nil en Go
user = None
```

---

## Colecciones

```python
from typing import Optional

# List — mutable, ordenada
products: list[str] = ["Laptop", "Mouse", "Teclado"]
mixed: list[int | str] = [1, "dos", 3]  # Python 3.10+ union con |

# Tuple — inmutable, ordenada
point: tuple[float, float] = (1.5, 2.7)
# Immutable → hasheable → puede ser key de dict

# Set — mutable, sin orden, sin duplicados
tags: set[str] = {"python", "backend", "fastapi"}
tags.add("docker")

# Frozenset — inmutable, hasheable
ALLOWED_ROLES: frozenset[str] = frozenset({"admin", "user", "viewer"})

# Dict — mutable, clave-valor
config: dict[str, str] = {
    "db_host": "localhost",
    "db_port": "5432",
}

# Acceso seguro
host = config.get("db_host", "localhost")  # default si no existe
port = config.get("db_missing")  # retorna None, no KeyError
```

---

## Type hints avanzados

```python
from typing import Optional, Union, Any, TypeVar, Generic

# Optional[X] = X | None (la forma antigua, aún válida)
def find_user(user_id: str) -> Optional[str]:
    ...

# Equivalente moderno (Python 3.10+)
def find_user(user_id: str) -> str | None:
    ...

# Union de tipos
def process(value: int | str | float) -> str:
    return str(value)

# TypeVar — para generics
T = TypeVar("T")

def first(items: list[T]) -> T | None:
    return items[0] if items else None

# Generic class
class Response(Generic[T]):
    def __init__(self, data: T, message: str = "OK"):
        self.data = data
        self.message = message

resp: Response[str] = Response(data="resultado", message="OK")
resp2: Response[list[int]] = Response(data=[1, 2, 3])

# TypedDict — dict con tipos fijos
from typing import TypedDict

class ProductDict(TypedDict):
    id: str
    name: str
    price: float

# Literal — valor específico
from typing import Literal
status: Literal["active", "inactive", "draft"] = "active"

# Final — constante que no debe reasignarse
from typing import Final
MAX_RETRIES: Final = 3
```

---

## Dataclasses — structs de Python

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Product:
    id: str
    name: str
    price: float
    stock: int = 0             # valor default
    tags: list[str] = field(default_factory=list)  # default mutable: SIEMPRE usar field
    created_at: datetime = field(default_factory=datetime.utcnow)

    # ¡Nunca hagas esto!
    # tags: list[str] = []  # todas las instancias compartirían la MISMA lista

product = Product(id="p1", name="Laptop", price=1500.0)
print(product)  # Product(id='p1', name='Laptop', price=1500.0, stock=0, ...)

# frozen=True → inmutable (como frozen struct en Rust)
@dataclass(frozen=True)
class ProductID:
    value: str

    def __post_init__(self):
        if not self.value:
            raise ValueError("ProductID no puede estar vacío")

pid = ProductID(value="abc-123")
# pid.value = "otro"  # DataclassError: cannot assign to field
```

---

## Variables y scope

```python
# LEGB rule: Local → Enclosing → Global → Built-in
x = "global"

def outer():
    x = "enclosing"

    def inner():
        # x es "enclosing" por closure
        print(x)  # "enclosing"

    inner()

# global — modificar variable global desde función (evitar)
counter = 0

def increment():
    global counter  # declarar intent explícitamente
    counter += 1

# nonlocal — modificar variable en función enclosing
def make_counter():
    count = 0

    def increment():
        nonlocal count
        count += 1
        return count

    return increment

counter = make_counter()
counter()  # 1
counter()  # 2
```

---

## Strings — operaciones útiles

```python
s = "  Hola Mundo  "

# Limpieza
s.strip()        # "Hola Mundo"
s.lstrip()       # "Hola Mundo  "
s.rstrip()       # "  Hola Mundo"

# Transformaciones
"hello world".title()   # "Hello World"
"Hello".upper()         # "HELLO"
"Hello".lower()         # "hello"

# Búsqueda
"python".startswith("py")  # True
"python".endswith("on")    # True
"hello world".find("world")  # 6 (índice)
"hello world".count("l")     # 3

# Split / Join
"a,b,c".split(",")           # ["a", "b", "c"]
", ".join(["a", "b", "c"])   # "a, b, c"

# Reemplazo
"hello world".replace("world", "python")  # "hello python"

# Format — la forma más segura para construcción de strings
# NUNCA concatenar input del usuario directamente en queries o comandos
name = "Rolando"
f"Hola, {name}"  # f-string — la más rápida y legible
"Hola, {}".format(name)  # .format()
"Hola, %s" % name  # %-formatting — estilo antiguo

# Multiline sin \n extra
query = (
    "SELECT * "
    "FROM products "
    "WHERE id = %s"
)
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Sin type hints — no sabés qué entra ni qué sale
def get_product(id):
    return db.get(id)

# Mutable default argument — bug silencioso
def add_tag(product, tags=[]):  # MAL: [] se comparte entre llamadas
    tags.append("new")
    return tags

# Concatenación de strings para queries → SQL injection
query = "SELECT * FROM products WHERE id = " + user_id  # PELIGROSO
```

### Profesional

```python
# Type hints en todo
def get_product(product_id: str) -> Optional[Product]:
    return db.get(product_id)

# Default factory para mutables
def add_tag(product: Product, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []
    tags.append("new")
    return tags

# Queries parametrizadas
cursor.execute("SELECT * FROM products WHERE id = %s", (user_id,))
```
