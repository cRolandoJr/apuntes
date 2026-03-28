# Structs e Interfaces

## Structs

Un struct es un conjunto de campos nombrados. Es el tipo de dato central en Go.

```go
type Product struct {
    ID          string
    Name        string
    Description string
    Price       float64
    Stock       int
    CreatedAt   time.Time
    Tags        []string
    Metadata    map[string]string
}

// Inicialización — siempre nombrar los campos
p := Product{
    ID:    "prod-001",
    Name:  "Teclado mecánico",
    Price: 89.99,
    Stock: 10,
}
// p.Description == ""    (zero value)
// p.CreatedAt == time.Time{} (zero value)
```

> Inicializar con nombres (no posicional) es obligatorio en código profesional. Si el struct agrega un campo, el código posicional deja de compilar — pero es una falla de mantenimiento, no de seguridad.

### Structs anónimos — para uso local

```go
// Útil en tests y serialización puntual
response := struct {
    Status  string `json:"status"`
    Message string `json:"message"`
}{
    Status:  "ok",
    Message: "producto creado",
}
```

---

## Struct tags — metadata para serialización

```go
type User struct {
    ID        string    `json:"id"         db:"id"`
    FirstName string    `json:"firstName"  db:"first_name"`
    Password  string    `json:"-"`         // omitir en JSON
    CreatedAt time.Time `json:"createdAt"  db:"created_at"`
    Score     *float64  `json:"score,omitempty"` // omitir si nil
}
```

---

## Struct Embedding — composición en vez de herencia

Go no tiene herencia. Usa composición mediante embedding.

```go
type Timestamps struct {
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt *time.Time // puntero para soft delete
}

type Product struct {
    ID    string
    Name  string
    Price float64
    Timestamps  // embedding — los campos de Timestamps son accesibles directamente
}

p := Product{
    ID:   "prod-001",
    Name: "Teclado",
    Timestamps: Timestamps{
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    },
}
fmt.Println(p.CreatedAt) // acceso directo al campo embebido
```

### Embedding de interfaces — composición de contratos

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter combina ambas interfaces
type ReadWriter interface {
    Reader
    Writer
}
```

---

## Interfaces

En Go las interfaces son implícitas. Un tipo satisface una interfaz si implementa todos sus métodos. No se declara que se "implementa" nada.

```go
type Animal interface {
    Sound() string
    Name() string
}

type Dog struct{ name string }
func (d Dog) Sound() string { return "Guau" }
func (d Dog) Name() string  { return d.name }

type Cat struct{ name string }
func (c Cat) Sound() string { return "Miau" }
func (c Cat) Name() string  { return c.name }

// Ambos satisfacen Animal sin declararlo
func Describe(a Animal) string {
    return fmt.Sprintf("%s dice %s", a.Name(), a.Sound())
}

Describe(Dog{name: "Rex"})  // "Rex dice Guau"
Describe(Cat{name: "Tom"})  // "Tom dice Miau"
```

---

## Interfaces pequeñas — el principio de Go

```go
// Mal — interfaz gigante que pocas cosas pueden implementar
type ProductRepository interface {
    GetAll() ([]*Product, error)
    GetByID(id string) (*Product, error)
    GetByName(name string) ([]*Product, error)
    GetByCategory(catID string) ([]*Product, error)
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
    BulkCreate(products []*Product) error
    BulkDelete(ids []string) error
    Count() (int, error)
    Search(query string) ([]*Product, error)
    // 10 métodos más...
}

// Bien — el usecase define SOLO lo que necesita
// (Interface Segregation — ver SOLID)
type ProductReader interface {
    GetByID(id string) (*Product, error)
    GetAll() ([]*Product, error)
}

type ProductWriter interface {
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
}

// En el paquete de dominio, la interfaz completa para el usecase de admin
type ProductRepository interface {
    ProductReader
    ProductWriter
}
```

---

## Type assertions y type switches

```go
// Type assertion — afirmar el tipo concreto de una interfaz
var a Animal = Dog{name: "Rex"}

dog, ok := a.(Dog)  // forma segura
if ok {
    fmt.Println("Es un perro:", dog.name)
}

// Forma insegura — panic si el tipo no es Dog
// dog := a.(Dog)  // NO hacer esto sin ok

// Type switch
func Describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("entero: %d", v)
    case string:
        return fmt.Sprintf("string: %s", v)
    case []int:
        return fmt.Sprintf("slice de ints con %d elementos", len(v))
    default:
        return fmt.Sprintf("tipo desconocido: %T", v)
    }
}
```

---

## La interfaz vacía — `any` / `interface{}`

```go
// interface{} acepta cualquier tipo (igual que any desde Go 1.18)
func PrintAnything(v any) {
    fmt.Println(v)
}

// En mapas de configuración
config := map[string]any{
    "host":    "localhost",
    "port":    8080,
    "debug":   true,
    "timeout": 30.5,
}
```

> Evitar `any` en código de dominio. Cuando lo ves, generalmente indica falta de tipado o diseño incompleto. Está bien en configuración, serialización o utilidades genéricas.

---

## Práctica: Novato vs Profesional

### Novato — dependencias concretas

```go
// El usecase depende del tipo concreto del repositorio
// Si cambiás de in-memory a PostgreSQL, hay que tocar el usecase
type ProductUseCase struct {
    Repo *MemoryProductRepository  // tipo concreto
}

func (u *ProductUseCase) GetByID(id string) (*Product, error) {
    return u.Repo.GetByID(id)
}
```

### Profesional — programar contra interfaces

```go
// El usecase solo conoce la interfaz, no la implementación
// Se puede testear con un mock, cambiar a PostgreSQL sin tocar el usecase

// domain/product.go — la interfaz vive en el dominio
type ProductRepository interface {
    GetByID(id string) (*Product, error)
    GetAll() ([]*Product, error)
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
}

// usecase/product_usecase.go — depende de la interfaz
type productUseCase struct {
    repo domain.ProductRepository  // interfaz
}

func NewProductUseCase(repo domain.ProductRepository) domain.ProductUseCase {
    return &productUseCase{repo: repo}
}

// Implementaciones (en paquetes externos al dominio):
// - repository/memory_repository.go  → MemoryProductRepository implements domain.ProductRepository
// - repository/postgres_repository.go → PostgresProductRepository implements domain.ProductRepository
// - En tests: MockProductRepository implements domain.ProductRepository
```

---

## Patrón: Check de interfaz en compile time

```go
// Verificar que un tipo implementa una interfaz sin esperar al runtime
// Si MemoryProductRepository no implementa la interfaz, hay error de compilación
var _ domain.ProductRepository = (*MemoryProductRepository)(nil)
```

Este patrón es un one-liner que se pone al principio del archivo de implementación. Muy útil cuando la interfaz tiene muchos métodos.

---

## Interfaces en la práctica — dónde definirlas

```
Donde VIVEN las interfaces: en el paquete del CONSUMIDOR (quien la usa)
NO en el paquete de quien la implementa

domain/product.go          <- define ProductRepository (la usa el usecase)
repository/memory.go       <- implementa ProductRepository
repository/postgres.go     <- también implementa ProductRepository
```

Esta es la diferencia entre Go y otros lenguajes. En Java/C# las interfaces las define quien implementa. En Go, las define quien usa — lo que permite desacoplar sin coordinación.
