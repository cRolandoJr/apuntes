# Clean Architecture en Go

## El problema que resuelve

Sin arquitectura, el código termina siendo una masa donde:

- La lógica de negocio está en los handlers HTTP
- Los handlers llaman directamente a la DB
- Testear requiere una DB real corriendo
- Cambiar el framework rompe la lógica de negocio
- Agregar una feature toca 5 archivos en capas mezcladas

Clean Architecture resuelve esto con una regla simple.

---

## La Dependency Rule (regla de dependencia)

```
Las dependencias solo pueden apuntar hacia adentro.
El código interior no conoce nada del código exterior.

Outer (Delivery/Infra)  →  UseCase  →  Domain
Repository              →  Domain
```

El **Domain** (centro) nunca importa nada de afuera. Es código Go puro.

```
┌─────────────────────────────────────────┐
│  Frameworks & Drivers                   │  ← PostgreSQL, HTTP, GraphQL, Redis
│  ┌──────────────────────────────────┐   │
│  │  Interface Adapters (Delivery)   │   │  ← handlers, resolvers, controllers
│  │  ┌───────────────────────────┐   │   │
│  │  │  Application (Use Cases)  │   │   │  ← lógica de aplicación
│  │  │  ┌────────────────────┐   │   │   │
│  │  │  │   Domain/Entities  │   │   │   │  ← reglas de negocio puras
│  │  │  └────────────────────┘   │   │   │
│  │  └───────────────────────────┘   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## Las capas y sus responsabilidades

### Domain (centro)

- Entidades: structs con reglas de negocio puras
- Interfaces: contratos que las capas exteriores deben implementar
- Errores de dominio tipados
- **NO importa**: ningún paquete externo al módulo (ni DB, ni HTTP, ni frameworks)

```go
// internal/domain/product.go
package domain

import "time"

type Product struct {
    ID          string
    Name        string
    Description string
    Price       float64
    Stock       int
    CreatedAt   time.Time
}

// Regla de negocio pura — no depende de nada externo
func (p *Product) IsAvailable() bool {
    return p.Stock > 0
}

func (p *Product) Apply(discountPercent float64) {
    p.Price = p.Price * (1 - discountPercent/100)
}

// Contrato para con el mundo exterior
type ProductRepository interface {
    GetAll() ([]*Product, error)
    GetByID(id string) (*Product, error)
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
}

type ProductUseCase interface {
    GetAll() ([]*Product, error)
    GetByID(id string) (*Product, error)
    Create(input CreateProductInput) (*Product, error)
    Update(id string, input UpdateProductInput) (*Product, error)
    Delete(id string) error
}
```

### Use Case (aplicación)

- Orquesta el flujo de una operación de negocio
- Valida reglas de negocio (no de formato)
- Depende SOLO del dominio (interfaces)
- **NO importa**: frameworks web, drivers de DB, paquetes de delivery

```go
// internal/usecase/product_usecase.go
package usecase

import (
    "fmt"
    "strings"
    "time"

    "gestion_productos/internal/domain"  // SOLO el dominio

    "github.com/google/uuid"
)

type productUseCase struct {
    repo domain.ProductRepository  // interfaz, no tipo concreto
}

func NewProductUseCase(repo domain.ProductRepository) domain.ProductUseCase {
    return &productUseCase{repo: repo}
}

func (u *productUseCase) Create(input domain.CreateProductInput) (*domain.Product, error) {
    if strings.TrimSpace(input.Name) == "" {
        return nil, domain.NewInvalidInputError("el nombre no puede estar vacío")
    }
    if input.Price <= 0 {
        return nil, domain.NewInvalidInputError("el precio debe ser mayor a cero")
    }

    product := &domain.Product{
        ID:          uuid.New().String(),
        Name:        input.Name,
        Description: input.Description,
        Price:       input.Price,
        Stock:       input.Stock,
        CreatedAt:   time.Now(),
    }

    if err := u.repo.Create(product); err != nil {
        return nil, fmt.Errorf("crear producto: %w", err)
    }

    return product, nil
}
```

### Repository (infraestructura de datos)

- Implementa las interfaces del dominio
- Habla con la base de datos
- Traduce errores de DB a errores de dominio
- **Importa**: dominio + driver de DB

```go
// internal/repository/postgres_repository.go
package repository

import (
    "context"
    "errors"

    "gestion_productos/internal/domain"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

type postgresProductRepository struct {
    db *pgxpool.Pool
}

func NewPostgresProductRepository(db *pgxpool.Pool) domain.ProductRepository {
    return &postgresProductRepository{db: db}
}

func (r *postgresProductRepository) GetByID(id string) (*domain.Product, error) {
    var p domain.Product
    err := r.db.QueryRow(context.Background(),
        "SELECT id, name, description, price, stock, created_at FROM products WHERE id = $1",
        id,
    ).Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.CreatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, domain.NewNotFoundError("producto no encontrado")
        }
        return nil, fmt.Errorf("repositorio GetByID: %w", err)
    }
    return &p, nil
}
```

### Delivery (entrega)

- Traduce entre el protocolo externo (HTTP, GraphQL, gRPC) y el dominio
- Llama al UseCase
- Maneja la presentación del error al cliente
- **Importa**: dominio + usecase + framework web

```go
// internal/delivery/graphql/resolver.go
package graphql

import (
    "context"
    "gestion_productos/internal/domain"
)

type Resolver struct {
    ProductUC domain.ProductUseCase
}

func (r *mutationResolver) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    return r.ProductUC.Create(input)  // solo delega al usecase
}
```

### cmd/main.go (composición)

- El único lugar que conoce todas las capas
- Construye las dependencias (Dependency Injection manual)
- Conecta todo

```go
// cmd/main.go
func main() {
    // 1. Infraestructura
    db := setupDB(os.Getenv("DATABASE_URL"))

    // 2. Repository — implementación concreta
    repo := repository.NewPostgresProductRepository(db)

    // 3. UseCase — recibe la interfaz
    productUC := usecase.NewProductUseCase(repo)

    // 4. Delivery — recibe el usecase
    resolver := &graphql.Resolver{ProductUC: productUC}

    // 5. Servidor
    srv := setupServer(resolver)
    log.Fatal(http.ListenAndServe(":8080", srv))
}
```

---

## Estructura de archivos del proyecto real

```
internal/
├── domain/
│   ├── product.go          ← Product struct + interfaces + CreateProductInput/UpdateProductInput
│   ├── category.go
│   └── errors.go           ← AppError, NewNotFoundError, NewInvalidInputError, etc.
├── usecase/
│   ├── product_usecase.go
│   └── product_usecase_test.go  ← tests sin DB real
├── repository/
│   ├── memory_repository.go     ← implementación en memoria (dev/tests)
│   └── postgres_repository.go   ← implementación real
└── delivery/
    ├── graphql/
    │   └── resolver.go
    └── http/
        └── handler.go
```

---

## Señales de violación de la Dependency Rule

```go
// RED FLAG 1: usecase importa repository
package usecase
import "gestion_productos/internal/repository"  // viola la regla

// RED FLAG 2: dominio importa http o db
package domain
import "net/http"     // viola la regla
import "database/sql" // viola la regla

// RED FLAG 3: handler con lógica de negocio
func CreateProductHandler(w http.ResponseWriter, r *http.Request) {
    // validaciones de negocio directamente en el handler
    if price <= 0 { http.Error(w, "precio inválido", 400) }
    if name == "" { http.Error(w, "nombre requerido", 400) }
    // SQL directo en el handler
    db.Exec("INSERT INTO products ...")
}

// RED FLAG 4: dominio con métodos que llaman a la DB
func (p *Product) Save() error {
    return db.Exec("INSERT INTO products ...")  // viola la regla
}
```

---

## Práctica: Novato vs Profesional

### Novato — todo en main.go

```go
func main() {
    db, _ := sql.Open("postgres", "...")

    http.HandleFunc("/products", func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" {
            var input struct{ Name string; Price float64 }
            json.NewDecoder(r.Body).Decode(&input)

            if input.Name == "" {
                http.Error(w, "nombre requerido", 400)
                return
            }
            // SQL directo en el handler
            db.Exec("INSERT INTO products (name, price) VALUES ($1, $2)", input.Name, input.Price)
            json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
        }
    })
    http.ListenAndServe(":8080", nil)
}
```

### Profesional — cada capa en su lugar

Flujo de request:

```
HTTP Request
    → delivery/http/handler.go (parsea JSON → CreateProductInput)
        → usecase/product_usecase.go (valida, crea el Product)
            → repository/postgres_repository.go (persiste en DB)
        ← retorna *Product con error tipado
    → delivery/http/handler.go (serializa product a JSON, mapea error a HTTP status)
← HTTP Response
```
