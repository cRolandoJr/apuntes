# Pydantic v2 — Validación y Serialización

Pydantic es el motor de validación de FastAPI. Define schemas como clases Python con type hints, y Pydantic valida los datos automáticamente. v2 (2023+) es hasta 20x más rápido que v1 porque usa Rust internamente.

---

## BaseModel básico

```python
from pydantic import BaseModel, Field, EmailStr, HttpUrl
from decimal import Decimal
from datetime import datetime
from uuid import UUID

class ProductCreate(BaseModel):
    name: str
    description: str | None = None
    price: Decimal
    stock: int = 0
    category_id: int

class ProductResponse(BaseModel):
    id: int
    name: str
    description: str | None
    price: Decimal
    stock: int
    category_id: int
    created_at: datetime
    updated_at: datetime

# Pydantic convierte automáticamente tipos compatibles
p = ProductCreate(name="Laptop", price="1500.00", stock="100", category_id="3")
print(p.price)  # Decimal('1500.00') — no str
print(p.stock)  # 100 — int, no str
```

---

## Field() — restricciones de validación

```python
from pydantic import BaseModel, Field
from decimal import Decimal

class ProductCreate(BaseModel):
    name: str = Field(
        min_length=3,
        max_length=200,
        description="Nombre del producto",
        examples=["Laptop Dell XPS 15"],
    )
    description: str | None = Field(
        default=None,
        max_length=5000,
    )
    price: Decimal = Field(
        gt=Decimal("0"),         # mayor que 0
        decimal_places=2,        # máximo 2 decimales
        description="Precio en ARS",
    )
    stock: int = Field(default=0, ge=0)   # mayor o igual a 0
    sku: str = Field(
        pattern=r"^[A-Z]{2}-\d{6}$",  # regex
        examples=["AB-123456"],
    )
    tags: list[str] = Field(default_factory=list, max_length=10)

class UserCreate(BaseModel):
    email: EmailStr                    # valida formato de email
    website: HttpUrl | None = None     # valida URL
    age: int = Field(ge=18, le=120)
```

---

## model_config — configuración del modelo

```python
from pydantic import BaseModel, ConfigDict
from datetime import datetime

class ProductResponse(BaseModel):
    model_config = ConfigDict(
        # Leer desde atributos de objetos (ORM models)
        from_attributes=True,
        # Validar también al asignar valores después de crear
        validate_assignment=True,
        # Pasar a minúsculas en JSON (snake_case → camelCase)
        # alias_generator=to_camel,
        # Poblar por nombre O alias
        populate_by_name=True,
        # Modo estricto: no coerce tipos (str → int falla)
        # strict=True,
        # Ejemplo para la documentación
        json_schema_extra={
            "example": {
                "id": 1,
                "name": "Laptop Dell XPS 15",
                "price": "1500.00",
            }
        },
    )

    id: int
    name: str
    price: float
    created_at: datetime
```

---

## Validadores de campo — `@field_validator`

```python
from pydantic import BaseModel, field_validator, ValidationInfo
import re

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_]{3,50}$", v):
            raise ValueError(
                "Solo letras, números y guión bajo. Entre 3 y 50 caracteres."
            )
        return v.lower()  # normalizar a minúsculas

    @field_validator("email")
    @classmethod
    def email_lowercase(cls, v: str) -> str:
        return v.lower().strip()

    @field_validator("password")
    @classmethod
    def password_strong(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Mínimo 8 caracteres")
        if not re.search(r"[A-Z]", v):
            raise ValueError("Debe incluir al menos una mayúscula")
        if not re.search(r"\d", v):
            raise ValueError("Debe incluir al menos un número")
        return v  # nunca almacenar la contraseña — el usecase la hashea

# Validador con modo='before' — se corre antes de la conversión de tipo
class ProductCreate(BaseModel):
    price: float

    @field_validator("price", mode="before")
    @classmethod
    def parse_price(cls, v):
        # Acepta "1.500,00" (formato europeo) o "1500.00"
        if isinstance(v, str):
            v = v.replace(".", "").replace(",", ".")
        return v
```

---

## Validadores de modelo — `@model_validator`

Para validaciones que involucran múltiples campos:

```python
from pydantic import BaseModel, model_validator
from datetime import date

class DateRangeFilter(BaseModel):
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def check_dates(self) -> "DateRangeFilter":
        if self.end_date < self.start_date:
            raise ValueError("end_date debe ser posterior a start_date")

        delta = (self.end_date - self.start_date).days
        if delta > 365:
            raise ValueError("El rango no puede superar 365 días")

        return self

class PasswordConfirm(BaseModel):
    password: str
    confirm_password: str

    @model_validator(mode="after")
    def passwords_match(self) -> "PasswordConfirm":
        if self.password != self.confirm_password:
            raise ValueError("Las contraseñas no coinciden")
        return self
```

---

## Modelos anidados

```python
class Address(BaseModel):
    street: str
    city: str
    state: str
    postal_code: str
    country: str = "AR"

class ContactInfo(BaseModel):
    email: EmailStr
    phone: str | None = None
    address: Address | None = None

class CustomerCreate(BaseModel):
    name: str
    contact: ContactInfo

# Validación anidada automática
customer = CustomerCreate(
    name="Juan",
    contact={
        "email": "juan@example.com",
        "address": {
            "street": "San Martín 123",
            "city": "Viedma",
            "state": "Río Negro",
            "postal_code": "8500",
        }
    }
)
# Cada nivel se valida con su propio modelo
print(customer.contact.address.city)  # "Viedma"
```

---

## Schemas de Request vs Response — separar siempre

```python
# Regla: nunca devolver el modelo de ORM directamente
# Tener schemas separados para create, update, y response

class ProductBase(BaseModel):
    """Campos comunes."""
    name: str = Field(min_length=3, max_length=200)
    description: str | None = None
    price: Decimal = Field(gt=0)

class ProductCreate(ProductBase):
    """Para POST — sin id ni timestamps."""
    category_id: int
    sku: str

class ProductUpdate(BaseModel):
    """Para PATCH — todos los campos opcionales."""
    name: str | None = Field(default=None, min_length=3, max_length=200)
    description: str | None = None
    price: Decimal | None = Field(default=None, gt=0)
    stock: int | None = Field(default=None, ge=0)

class ProductResponse(ProductBase):
    """Para respuestas — tiene id y timestamps, nunca password_hash."""
    model_config = ConfigDict(from_attributes=True)

    id: int
    sku: str
    stock: int
    created_at: datetime
    updated_at: datetime
    category: "CategoryResponse"  # nested response

class ProductList(BaseModel):
    """Respuesta paginada."""
    items: list[ProductResponse]
    total: int
    page: int
    limit: int
    has_next: bool
```

---

## Serialización — `model_dump()` y `model_dump_json()`

```python
product = ProductResponse(
    id=1,
    name="Laptop",
    price=Decimal("1500.00"),
    created_at=datetime.now(),
    updated_at=datetime.now(),
)

# A dict
data = product.model_dump()
# {"id": 1, "name": "Laptop", "price": Decimal("1500.00"), ...}

# Solo ciertos campos
partial = product.model_dump(include={"id", "name", "price"})

# Excluir campos
safe = product.model_dump(exclude={"password_hash", "internal_notes"})

# A JSON string (usa el serializer interno de Pydantic — más rápido que json.dumps)
json_str = product.model_dump_json()

# Excluir None
clean = product.model_dump(exclude_none=True)

# Crear instancia desde otro modelo (update parcial)
update_data = product_update.model_dump(exclude_none=True)
for field, value in update_data.items():
    setattr(db_product, field, value)
```

---

## Validar datos externos

```python
# Desde dict (request body ya parseado)
try:
    product = ProductCreate.model_validate(raw_dict)
except ValidationError as e:
    for error in e.errors():
        print(error["loc"], error["msg"])  # campo, mensaje

# Desde JSON string
product = ProductCreate.model_validate_json(json_bytes)

# Desde ORM model (from_attributes=True debe estar en model_config)
db_product = await session.get(ProductModel, 1)
response = ProductResponse.model_validate(db_product)
```

---

## Práctica: Novato vs Profesional

### Novato

```python
# Un solo schema para todo — expone datos internos en responses
class Product(BaseModel):
    id: int | None = None
    name: str
    price: float
    password_hash: str | None = None   # MAL — se expone en respuesta
    internal_cost: float | None = None  # MAL — dato privado

# Sin validación — acepta cualquier precio
class OrderCreate(BaseModel):
    product_id: int
    quantity: int    # sin restricciones — puede ser -5 o 0
    total: float     # sin restricciones — puede ser negativo
```

### Profesional

```python
# Schemas separados por propósito
class OrderCreate(BaseModel):
    product_id: int = Field(gt=0)
    quantity: int = Field(ge=1, le=1000)

    # total se CALCULA en el usecase, no lo recibe el cliente
    # Así evitamos que alguien envíe total=0 para obtener gratis

class OrderResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    product_id: int
    quantity: int
    unit_price: Decimal
    total: Decimal
    status: OrderStatus
    created_at: datetime
    # SIN campos sensibles, SIN internals
```
