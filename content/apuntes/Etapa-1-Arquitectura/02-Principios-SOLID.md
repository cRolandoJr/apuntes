# Principios SOLID en Go

Los 5 principios para escribir código mantenible, extensible y testeable. En Go se expresan diferente que en Java pero aplican igual.

---

## S — Single Responsibility Principle

> Una clase/struct/función debería tener una sola razón para cambiar.

```go
// MALO — ProductService hace todo
type ProductService struct{ db *sql.DB }

func (s *ProductService) Create(input CreateInput) error { /* SQL directo */ }
func (s *ProductService) SendWelcomeEmail(p *Product) error { /* SMTP */ }
func (s *ProductService) ExportToCSV(products []*Product) ([]byte, error) { /* CSV */ }
func (s *ProductService) LogActivity(action string) { /* logging */ }

// Razones para cambiar: cambiar la DB, cambiar el email provider, cambiar el formato CSV,
// cambiar la lógica de logging... son 4 razones, viola SRP.
```

```go
// BIEN — cada struct tiene una responsabilidad
type productUseCase struct {
    repo         domain.ProductRepository
    notifier     domain.Notifier      // interfaz — no importa si es email, SMS, etc.
    exporter     domain.Exporter
}

func (u *productUseCase) Create(input domain.CreateProductInput) (*domain.Product, error) {
    // solo lógica de negocio
}

type emailNotifier struct{ smtp *smtpClient }
func (n *emailNotifier) Notify(event domain.Event) error { /* solo SMTP */ }

type csvExporter struct{}
func (e *csvExporter) Export(products []*domain.Product) ([]byte, error) { /* solo CSV */ }
```

---

## O — Open/Closed Principle

> El código debería estar abierto a extensión y cerrado a modificación.
> Agregar comportamiento nuevo sin tocar el código existente.

```go
// MALO — cada nuevo tipo de descuento requiere modificar la función
func CalculateDiscount(product *Product, discountType string) float64 {
    switch discountType {
    case "percentage":
        return product.Price * 0.1
    case "fixed":
        return 10.0
    case "seasonal":  // se agrega un nuevo tipo → hay que modificar esta función
        return product.Price * 0.15
    }
    return 0
}
```

```go
// BIEN — agregar un nuevo tipo de descuento = agregar un struct nuevo, no modificar nada
type DiscountStrategy interface {
    Calculate(price float64) float64
}

type PercentageDiscount struct{ Percent float64 }
func (d PercentageDiscount) Calculate(price float64) float64 {
    return price * (d.Percent / 100)
}

type FixedDiscount struct{ Amount float64 }
func (d FixedDiscount) Calculate(price float64) float64 {
    return d.Amount
}

type SeasonalDiscount struct{ Multiplier float64 } // nuevo — no toca nada existente
func (d SeasonalDiscount) Calculate(price float64) float64 {
    return price * d.Multiplier
}

func ApplyDiscount(product *Product, discount DiscountStrategy) float64 {
    return product.Price - discount.Calculate(product.Price)
}

// Uso
ApplyDiscount(product, PercentageDiscount{Percent: 10})
ApplyDiscount(product, SeasonalDiscount{Multiplier: 0.15})
```

---

## L — Liskov Substitution Principle

> Los tipos que implementan una interfaz deben poder sustituirse entre sí sin que el comportamiento del programa cambie.
> Si tenés una función que acepta una interfaz, cualquier implementación debería funcionar correctamente.

```go
type ProductRepository interface {
    GetByID(id string) (*Product, error)
    Create(p *Product) error
}

// MemoryRepository — para tests y dev
type MemoryRepository struct { products map[string]*Product }
func (r *MemoryRepository) GetByID(id string) (*Product, error) { ... }
func (r *MemoryRepository) Create(p *Product) error { ... }

// PostgresRepository — para producción
type PostgresRepository struct { db *pgxpool.Pool }
func (r *PostgresRepository) GetByID(id string) (*Product, error) { ... }
func (r *PostgresRepository) Create(p *Product) error { ... }

// El usecase funciona igual con ambas implementaciones
uc := NewProductUseCase(MemoryRepository{})    // para tests
uc := NewProductUseCase(PostgresRepository{})  // para producción
```

**Violación de LSP:**

```go
type ReadOnlyRepository struct{}
func (r *ReadOnlyRepository) GetByID(id string) (*Product, error) { ... }
func (r *ReadOnlyRepository) Create(p *Product) error {
    return errors.New("esta implementación no soporta Create")  // VIOLA LSP
    // la interfaz promete que Create funciona, pero esta implementación lo rompe
}
```

---

## I — Interface Segregation Principle

> Los clientes no deberían depender de interfaces que no usan.
> Interfaces pequeñas y enfocadas son mejores que interfaces grandes.

```go
// MALO — interfaz gigante
type ProductRepository interface {
    GetAll() ([]*Product, error)
    GetByID(id string) (*Product, error)
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
    BulkCreate(products []*Product) error
    Count() (int, error)
    Search(query string) ([]*Product, error)
    Archive(id string) error
    // ...más métodos...
}

// Si un usecase solo necesita leer, sigue necesitando depender de todos los métodos
```

```go
// BIEN — segregar según el caso de uso
type ProductReader interface {
    GetAll() ([]*Product, error)
    GetByID(id string) (*Product, error)
}

type ProductWriter interface {
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
}

type ProductSearcher interface {
    Search(query string) ([]*Product, error)
}

// Cada usecase declara solo lo que necesita
type productQueryUseCase struct {
    repo ProductReader  // solo lectura
}

type productCommandUseCase struct {
    repo interface {    // solo escritura
        ProductWriter
    }
}

// La implementación concreta implementa todas las interfaces relevantes
type postgresRepo struct{ db *pgxpool.Pool }
// ... implementa GetAll, GetByID, Create, Update, Delete, Search
// Satisface ProductReader, ProductWriter, y ProductSearcher implícitamente
```

---

## D — Dependency Inversion Principle

> Los módulos de alto nivel no deben depender de los de bajo nivel.
> Ambos deben depender de abstracciones (interfaces).
> Las abstracciones no deben depender de los detalles. Los detalles deben depender de las abstracciones.

Este es el principio más importante en Clean Architecture. Ya lo vemos en todo el código.

```go
// MALO — alto nivel (usecase) depende de bajo nivel (implementación concreta)
type productUseCase struct {
    repo *postgres.ProductRepository  // tipo concreto de bajo nivel
}

// Problema: si cambiás a MongoDB, tenés que tocar el usecase

// BIEN — usecase depende de la abstracción (interfaz)
type productUseCase struct {
    repo domain.ProductRepository  // interfaz — abstracción
}

// La inversión: quien define el contrato es el alto nivel (domain/usecase)
// quien lo implementa es el bajo nivel (postgres, memory, mongodb)
// El bajo nivel depende del contrato del alto nivel, no al revés
```

```
Sin DIP:   UseCase → PostgresRepo
Con DIP:   UseCase → <<interface>> ProductRepository ← PostgresRepo
                                                      ← MongoRepo
                                                      ← MemoryRepo (para tests)
```

---

## SOLID en la práctica — resumen rápido

Al leer código ajeno, preguntarte:

| Principio | Pregunta de diagnóstico                                                                             |
| --------- | --------------------------------------------------------------------------------------------------- |
| SRP       | ¿Cuántas razones tiene este archivo para cambiar? ¿Más de 1? → Separar                              |
| OCP       | ¿Para agregar un nuevo caso tengo que tocar código existente? → Usar interfaz/estrategia            |
| LSP       | ¿Alguna implementación lanza excepciones o retorna errores inesperados para métodos de la interfaz? |
| ISP       | ¿Algún struct depende de métodos de una interfaz que nunca usa? → Separar la interfaz               |
| DIP       | ¿Hay un import de un tipo concreto (repositorio, framework) en el usecase o domain? → Usar interfaz |
