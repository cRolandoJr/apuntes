# Clases y OOP en Python

Python es multiparadigma — OOP, funcional, procedural. En backend profesional se usa OOP para servicios, repositorios, y modelos de dominio.

---

## Clases — anatomía completa

```python
from __future__ import annotations  # permite forward references en type hints
from typing import ClassVar

class Product:
    # Class variable — compartida por TODAS las instancias
    DEFAULT_CATEGORY: ClassVar[str] = "general"
    _registry: ClassVar[dict[str, Product]] = {}

    def __init__(self, id: str, name: str, price: float, stock: int = 0) -> None:
        # Instance variables — propias de cada instancia
        self.id = id
        self.name = name
        self._price = price     # _ = convención "privado" (no enforced)
        self.__stock = stock    # __ = name mangling (_Product__stock)

    # Representación para debugging
    def __repr__(self) -> str:
        return f"Product(id={self.id!r}, name={self.name!r}, price={self._price})"

    # Representación legible para el usuario
    def __str__(self) -> str:
        return f"{self.name} — ${self._price:.2f}"

    # Igualdad
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Product):
            return NotImplemented
        return self.id == other.id

    # Para usar como key en dict o en set (requiere __eq__)
    def __hash__(self) -> int:
        return hash(self.id)

    # Comparación — permite sorted(), min(), max()
    def __lt__(self, other: Product) -> bool:
        return self._price < other._price

    @property
    def price(self) -> float:
        return self._price

    @price.setter
    def price(self, value: float) -> None:
        if value < 0:
            raise ValueError(f"Precio inválido: {value}")
        self._price = value

    @classmethod
    def from_dict(cls, data: dict) -> Product:
        """Factory method."""
        return cls(
            id=data["id"],
            name=data["name"],
            price=float(data["price"]),
            stock=data.get("stock", 0),
        )

    @staticmethod
    def validate_price(price: float) -> bool:
        return price > 0
```

---

## Dunder methods (Magic Methods) — los más útiles

```python
class Money:
    def __init__(self, amount: float, currency: str = "USD"):
        self.amount = amount
        self.currency = currency

    # Operadores aritméticos
    def __add__(self, other: Money) -> Money:
        if self.currency != other.currency:
            raise ValueError("no se pueden sumar distintas monedas")
        return Money(self.amount + other.amount, self.currency)

    def __mul__(self, factor: float) -> Money:
        return Money(self.amount * factor, self.currency)

    # Context manager — para "with" statements
    def __enter__(self) -> Money:
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        # exc_type = None si no hubo excepción
        # retornar True suprime la excepción
        return False

    # Longitud
    def __len__(self) -> int:
        return 1  # siempre 1 unidad monetaria

    # Iteración
    def __iter__(self):
        yield self.amount
        yield self.currency

    # Indexación
    def __getitem__(self, key: str) -> float | str:
        return {"amount": self.amount, "currency": self.currency}[key]

    # Llamable como función
    def __call__(self, factor: float) -> Money:
        return Money(self.amount * factor, self.currency)

# Context manager personalizado
class DBTransaction:
    def __init__(self, session):
        self.session = session

    def __enter__(self):
        return self.session

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.session.commit()
        else:
            self.session.rollback()
        return False  # no suprimir excepciones

with DBTransaction(db_session) as session:
    session.add(product)
    # si hay excepción → rollback automático
    # si no → commit automático
```

---

## Herencia — las reglas

```python
from abc import ABC, abstractmethod

# Clase abstracta — no puede instanciarse directamente
class Repository(ABC):
    @abstractmethod
    def get_by_id(self, id: str):
        ...

    @abstractmethod
    def save(self, entity) -> None:
        ...

    # Método concreto en clase abstracta — hereda comportamiento
    def exists(self, id: str) -> bool:
        return self.get_by_id(id) is not None

# Implementación concreta
class PostgresProductRepository(Repository):
    def __init__(self, session):
        self.session = session

    def get_by_id(self, id: str) -> Product | None:
        return self.session.query(Product).filter(Product.id == id).first()

    def save(self, product: Product) -> None:
        self.session.merge(product)
        self.session.commit()

# Herencia múltiple — Python permite MRO (Method Resolution Order)
class Auditable:
    def audit(self, action: str) -> None:
        print(f"Auditing: {action}")

class AuditableRepository(Repository, Auditable):
    # MRO: AuditableRepository → Repository → Auditable → ABC → object
    ...
```

### super() en herencia

```python
class BaseModel:
    def __init__(self, id: str) -> None:
        self.id = id
        self.created_at = datetime.utcnow()

    def to_dict(self) -> dict:
        return {"id": self.id, "created_at": self.created_at.isoformat()}

class Product(BaseModel):
    def __init__(self, id: str, name: str, price: float) -> None:
        super().__init__(id)  # llamar el __init__ del padre
        self.name = name
        self.price = price

    def to_dict(self) -> dict:
        base = super().to_dict()  # dict del padre
        base.update({"name": self.name, "price": self.price})
        return base
```

---

## Protocolos — duck typing formal (Python 3.8+)

Similar a las interfaces de Go. No requieren herencia — si el objeto tiene los métodos, es compatible.

```python
from typing import Protocol, runtime_checkable

# Definir el protocolo (interfaz)
@runtime_checkable  # permite isinstance() checking
class Persistable(Protocol):
    def save(self) -> None: ...
    def delete(self) -> None: ...

# Cualquier clase con estos métodos es compatible
# SIN necesidad de herencia explícita
class Product:
    def save(self) -> None:
        print("guardando producto")

    def delete(self) -> None:
        print("eliminando producto")

class User:
    def save(self) -> None:
        print("guardando usuario")

    def delete(self) -> None:
        print("eliminando usuario")

# Ambos son Persistable sin declararlo
def persist_all(items: list[Persistable]) -> None:
    for item in items:
        item.save()

persist_all([Product(), User()])  # funciona

# Verificar en runtime (solo con @runtime_checkable)
isinstance(Product(), Persistable)  # True
```

---

## Dataclasses vs Pydantic vs NamedTuple

```python
# dataclass — mutable, sin validación
@dataclass
class ProductDC:
    name: str
    price: float

p = ProductDC(name="Laptop", price="1500")  # NO valida, acepta string para price

# NamedTuple — inmutable, sin validación, más liviana
from typing import NamedTuple
class Point(NamedTuple):
    x: float
    y: float

pt = Point(1.0, 2.0)
pt.x  # 1.0

# Pydantic BaseModel — validación completa (ver [[02-Pydantic-y-Validacion]])
from pydantic import BaseModel

class ProductPydantic(BaseModel):
    name: str
    price: float  # convierte "1500" → 1500.0, falla si no puede convertir

p = ProductPydantic(name="Laptop", price="1500")  # OK, convierte
p = ProductPydantic(name="Laptop", price="abc")   # ValidationError
```

**Cuándo usar cada uno:**

- `dataclass`: modelos de dominio internos, sin necesidad de validación de entrada externa
- `NamedTuple`: cuando necesitás inmutabilidad y menor overhead
- `Pydantic BaseModel`: validación de input externo (API requests, env vars, config files)

---

## Práctica: Novato vs Profesional

### Novato

```python
# Sin __repr__ — debugging imposible
class Product:
    def __init__(self, id, name, price):
        self.id = id
        self.name = name
        self.price = price

p = Product("1", "Laptop", 1500)
print(p)  # <__main__.Product object at 0x7f1234>

# Mutando estado desde afuera sin validación
p.price = -100  # nadie lo impide
```

### Profesional

```python
@dataclass
class Product:
    id: str
    name: str
    _price: float = field(repr=False)

    @property
    def price(self) -> float:
        return self._price

    @price.setter
    def price(self, v: float) -> None:
        if v < 0:
            raise ValueError(f"precio inválido: {v}")
        self._price = v

    def __repr__(self) -> str:
        return f"Product(id={self.id!r}, name={self.name!r}, price={self._price})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Product):
            return NotImplemented
        return self.id == other.id
```
