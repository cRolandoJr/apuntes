# ORM y Transacciones en Go

Un ORM (Object-Relational Mapper) traduce entre el mundo de objetos de tu código y el mundo relacional de la base de datos. Abstrae el SQL, maneja relaciones, ciclo de vida de entidades y transacciones.

---

## Active Record vs Data Mapper — dos filosofías

### Active Record (el modelo se persiste a sí mismo)

```ruby
# Ruby on Rails — el modelo ES la persistencia
product = Product.new(name: "Laptop", price: 1500)
product.save  # el objeto sabe cómo guardarse
product.destroy
```

En Go no hay herencia ni métodos de instancia en structs heredados, entonces los ORMs en Go implementan variantes de esto. GORM usa un enfoque híbrido.

### Data Mapper (separación entre dominio y persistencia)

```go
// El struct de dominio NO sabe nada de la DB
type Product struct {
    ID    string
    Name  string
    Price float64
}

// El mapper (repositorio) maneja la persistencia
type ProductRepository struct {
    db *gorm.DB
}

func (r *ProductRepository) Save(p *Product) error {
    return r.db.Save(p).Error
}
```

**En Go profesional se usa Data Mapper** porque respeta Clean Architecture: el dominio no tiene dependencias de infraestructura.

---

## El patrón Unit of Work

Un UoW rastrea todos los cambios a entidades durante una "unidad de trabajo" y los escribe en la base de datos en una sola transacción al final.

```
Inicio de UoW
  ↓
Cargar Product (tracked como "clean")
  ↓
Modificar product.Price (tracked como "dirty")
  ↓
Crear nueva Order (tracked como "new")
  ↓
Commit → UoW genera:
  UPDATE products SET price = ... WHERE id = ...
  INSERT INTO orders (...)
  Todo en una TRANSACCIÓN
```

Los ORMs implementan esto implícitamente en transacciones. En Go, una transacción `db.Begin()` actúa como el Unit of Work.

---

## ORMs en Go — comparación

| ORM      | Filosofía            | Fortaleza                                                            | Debilidad                            |
| -------- | -------------------- | -------------------------------------------------------------------- | ------------------------------------ |
| **GORM** | Data Mapper híbrido  | Maduro, ecosistema grande, documentación excelente                   | Magia implícita, performance en bulk |
| **Bun**  | Data Mapper          | Performance, API similar a `database/sql`, soporte de `database/sql` | Menor ecosistema                     |
| **Ent**  | Code generation      | Type-safe al 100%, esquema en Go, traversals                         | Curva de aprendizaje alta            |
| **sqlx** | Query builder/mapper | Control total del SQL, liviano                                       | No es ORM real, manual               |

**Cuándo usar qué:**

- GORM: proyectos nuevos que quieren productividad, equipo sin mucha experiencia en SQL
- Bun: performance es crítica, querés escribir SQL explícito pero con ayuda
- Ent: grafos de entidades complejos, type-safety extrema
- sqlx o pgx puro: máximo control, queries complejas que un ORM generaría mal

---

## GORM — Setup completo

```bash
go get gorm.io/gorm
go get gorm.io/driver/postgres
```

```go
// internal/infra/database/gorm.go

import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func NewGORMDB(dsn string, env string) (*gorm.DB, error) {
    logLevel := logger.Silent
    if env != "production" {
        logLevel = logger.Info  // muestra las queries SQL en desarrollo
    }

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logLevel),
        // NamingConvention: convention que convierte CamelCase → snake_case
        // Product.CreatedAt → products.created_at (automático)

        // IMPORTANTE: deshabilita transacciones automáticas en Create/Update/Delete
        // para mejor performance cuando manejás tus propias transacciones
        SkipDefaultTransaction: false,

        // Preparar statements para queries repetidas → mejor performance
        PrepareStmt: true,
    })
    if err != nil {
        return nil, fmt.Errorf("conectar GORM: %w", err)
    }

    // Configurar el pool de conexiones subyacente (database/sql)
    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db, nil
}
```

---

## Modelos GORM — convenciones

```go
// GORM usa convenciones para mapear automáticamente
// Product → tabla "products" (plural, snake_case)

type Product struct {
    // gorm.Model incluye: ID (uint), CreatedAt, UpdatedAt, DeletedAt (soft delete)
    gorm.Model

    // Campos propios
    Name        string          `gorm:"not null;size:255"`
    Description string          `gorm:"default:''"`
    Price       decimal.Decimal `gorm:"type:decimal(12,2);not null"`
    Stock       int             `gorm:"not null;default:0"`

    // Foreign key — GORM infiere: CategoryID → categories.id
    CategoryID uint
    Category   Category `gorm:"foreignKey:CategoryID"`

    // Índices
    SKU         string `gorm:"uniqueIndex;not null"`
    Name_       string `gorm:"index:idx_name_category,priority:1"`

    // Tags útiles
    // column          → nombre de columna custom
    // type            → tipo SQL explícito
    // not null        → constraint NOT NULL
    // uniqueIndex     → índice único
    // index           → índice normal
    // default         → valor por defecto
    // ->:false        → solo escritura (no se lee/escanea)
    // <-:false        → solo lectura (no se escribe)
    // -               → ignorar el campo
}

type Category struct {
    gorm.Model
    Name     string    `gorm:"not null;uniqueIndex"`
    Products []Product `gorm:"foreignKey:CategoryID"` // HasMany
}
```

### Usar uuid en lugar de uint

```go
import "github.com/google/uuid"

type Product struct {
    ID          string `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    Name        string
    CategoryID  string `gorm:"type:uuid;not null"`
    Category    Category
    CreatedAt   time.Time
    UpdatedAt   time.Time
    DeletedAt   gorm.DeletedAt `gorm:"index"` // soft delete manual
}
```

---

## CRUD básico

```go
// CREATE
product := &Product{Name: "Laptop", Price: 1500.0, CategoryID: "cat-id"}
result := db.Create(product)
// product.ID ya tiene el ID generado por PostgreSQL
if result.Error != nil {
    return result.Error
}
fmt.Println("Rows affected:", result.RowsAffected)

// READ — First vs Find vs Take
var p Product

// First: ORDER BY id ASC LIMIT 1 + error si no encuentra
db.First(&p, "id = ?", "some-id")

// Take: sin ORDER BY LIMIT 1 + error si no encuentra (más rápido)
db.Take(&p, "id = ?", "some-id")

// Find: no falla si no encuentra, retorna slice vacío
var products []Product
db.Find(&products)  // todos
db.Where("price > ?", 100).Find(&products)

// Manejo de ErrRecordNotFound
if err := db.First(&p, id).Error; err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, domain.NewNotFoundError("producto no encontrado")
    }
    return nil, fmt.Errorf("buscar producto: %w", err)
}

// UPDATE — Save vs Updates
// Save: actualiza TODOS los campos (incluyendo zero values)
p.Price = 1400.0
db.Save(&p)  // UPDATE products SET name=..., price=1400, ... WHERE id=...

// Updates: actualiza solo los campos especificados
db.Model(&p).Updates(map[string]interface{}{
    "price": 1400.0,
    "stock": 0,
})

// Update con struct (solo actualiza campos non-zero)
db.Model(&p).Updates(Product{Price: 1400.0})
// Si Price es 0.0 (zero value de float), NO lo actualiza — usar map para zero values

// DELETE — soft delete con gorm.Model o DeletedAt
db.Delete(&p)
// UPDATE products SET deleted_at = NOW() WHERE id = ...

// Hard delete
db.Unscoped().Delete(&p)
// DELETE FROM products WHERE id = ...

// Los Find/First/etc. ignoran registros con deleted_at != NULL automáticamente
// Para incluirlos: db.Unscoped().Find(&products)
```

---

## Transacciones en GORM

### Automática con closure — la forma idiomática

```go
func (r *gormProductRepo) CreateWithInventory(
    ctx context.Context,
    product *domain.Product,
    adjustment domain.InventoryAdjustment,
) (*domain.Product, error) {
    var result *Product

    // db.Transaction ejecuta la función en una transacción
    // Si retorna un error → ROLLBACK automático
    // Si retorna nil → COMMIT automático
    err := r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        // Usar tx (no r.db) para todas las operaciones dentro de la transacción
        gormProduct := toGORMProduct(product)
        if err := tx.Create(gormProduct).Error; err != nil {
            return fmt.Errorf("crear producto: %w", err)  // → ROLLBACK
        }

        gormAdj := &InventoryAdjustment{
            ProductID: gormProduct.ID,
            Quantity:  adjustment.Quantity,
            Reason:    adjustment.Reason,
        }
        if err := tx.Create(gormAdj).Error; err != nil {
            return fmt.Errorf("crear ajuste: %w", err)  // → ROLLBACK
        }

        result = gormProduct
        return nil  // → COMMIT
    })

    if err != nil {
        return nil, err
    }
    return toDomainProduct(result), nil
}
```

### Manual — cuando necesitás más control

```go
func (r *gormProductRepo) TransferStock(ctx context.Context, fromID, toID string, qty int) error {
    tx := r.db.WithContext(ctx).Begin()
    if tx.Error != nil {
        return fmt.Errorf("iniciar transacción: %w", tx.Error)
    }

    // defer rollback — si no se llamó Commit(), hace Rollback()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
        }
    }()

    // Bloquear las filas para evitar race conditions
    var from, to Product
    if err := tx.Set("gorm:query_option", "FOR UPDATE").First(&from, "id = ?", fromID).Error; err != nil {
        tx.Rollback()
        return err
    }
    if err := tx.Set("gorm:query_option", "FOR UPDATE").First(&to, "id = ?", toID).Error; err != nil {
        tx.Rollback()
        return err
    }

    if from.Stock < qty {
        tx.Rollback()
        return domain.NewInvalidInputError("stock insuficiente")
    }

    if err := tx.Model(&from).Update("stock", gorm.Expr("stock - ?", qty)).Error; err != nil {
        tx.Rollback()
        return err
    }
    if err := tx.Model(&to).Update("stock", gorm.Expr("stock + ?", qty)).Error; err != nil {
        tx.Rollback()
        return err
    }

    return tx.Commit().Error
}
```

### Nested Transactions con SavePoints

```go
// SavePoints permiten rollback parcial dentro de una transacción
err := db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&product)

    // SavePoint — "hasta acá fue bien"
    tx.SavePoint("before_inventory")

    if err := tx.Create(&inventory).Error; err != nil {
        // Solo revierta lo que pasó después del SavePoint
        tx.RollbackTo("before_inventory")
        // El producto SÍ fue creado, el inventario no
        // Continúa la transacción sin el inventory
        log.Warn("no se pudo crear inventario inicial, continuando...")
    }

    tx.Create(&audit)
    return nil
})
```

---

## Associations — relaciones

```go
// HasOne: User tiene un Profile (FK en profiles.user_id)
type User struct {
    gorm.Model
    Email   string
    Profile Profile  // GORM infiere FK: profiles.user_id
}
type Profile struct {
    gorm.Model
    UserID uint  // FK explícita
    Bio    string
}

// HasMany: Category tiene muchos Products
type Category struct {
    gorm.Model
    Name     string
    Products []Product  // FK: products.category_id
}

// BelongsTo: Product pertenece a Category
type Product struct {
    gorm.Model
    CategoryID uint
    Category   Category  // BelongsTo — FK en products misma tabla
}

// ManyToMany: Product tiene muchos Tags, Tag pertenece a muchos Products
type Product struct {
    gorm.Model
    Tags []Tag `gorm:"many2many:product_tags;"` // tabla join: product_tags
}
type Tag struct {
    gorm.Model
    Name     string
    Products []Product `gorm:"many2many:product_tags;"`
}
```

---

## N+1 en GORM — Preload vs Joins

```go
// SIN solución — N+1 queries
var products []Product
db.Find(&products)
for _, p := range products {
    // Acceder a p.Category dispara una query por producto
    fmt.Println(p.Category.Name)  // 1 query por iteración → N+1
}

// SOLUCIÓN 1: Preload — 2 queries (recomendado para HasMany, M2M)
db.Preload("Category").Find(&products)
// SELECT * FROM products
// SELECT * FROM categories WHERE id IN (1, 2, 3, ...)

// Preload con condición
db.Preload("Products", "stock > ?", 0).Find(&categories)

// Preload anidado
db.Preload("Category.Parent").Find(&products)

// SOLUCIÓN 2: Joins — 1 query (recomendado para HasOne, BelongsTo)
db.Joins("Category").Find(&products)
// SELECT products.*, categories.* FROM products
// LEFT JOIN categories ON products.category_id = categories.id

// Preload múltiple
db.Preload("Category").Preload("Tags").Find(&products)
```

---

## Hooks — ciclo de vida

```go
type Product struct {
    gorm.Model
    Name  string
    Slug  string `gorm:"uniqueIndex"`
    Price float64
}

// Se ejecuta ANTES del INSERT — para hashear passwords, generar slugs, etc.
func (p *Product) BeforeCreate(tx *gorm.DB) error {
    p.Slug = slugify(p.Name)  // "Laptop Pro" → "laptop-pro"
    if p.Price <= 0 {
        return errors.New("el precio debe ser mayor a 0")  // → ROLLBACK
    }
    return nil
}

// Se ejecuta DESPUÉS del INSERT
func (p *Product) AfterCreate(tx *gorm.DB) error {
    // Actualizar cache, enviar evento, etc.
    return nil
}

// Otros hooks: BeforeUpdate, AfterUpdate, BeforeDelete, AfterDelete,
// BeforeSave (Create + Update), AfterSave, AfterFind
```

---

## Scopes — queries reutilizables

```go
// Definir scopes como funciones
func InStock(db *gorm.DB) *gorm.DB {
    return db.Where("stock > 0")
}

func ByCategory(categoryID string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("category_id = ?", categoryID)
    }
}

func PriceRange(min, max float64) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("price BETWEEN ? AND ?", min, max)
    }
}

func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

// Usar scopes — se combinan limpiamente
db.Scopes(InStock, ByCategory("cat-id"), PriceRange(100, 500), Paginate(2, 20)).
    Preload("Category").
    Find(&products)
```

---

## Repository Pattern con GORM

```go
// El dominio define la interfaz — NO sabe de GORM
type ProductRepository interface {
    GetAll(ctx context.Context, filter ProductFilter) ([]*Product, int64, error)
    GetByID(ctx context.Context, id string) (*Product, error)
    Create(ctx context.Context, p *Product) (*Product, error)
    Update(ctx context.Context, p *Product) (*Product, error)
    Delete(ctx context.Context, id string) error
}

// La implementación GORM está en infra/ — desacoplada del dominio
type gormProductRepository struct {
    db *gorm.DB
}

func NewGORMProductRepository(db *gorm.DB) ProductRepository {
    return &gormProductRepository{db: db}
}

func (r *gormProductRepository) GetByID(ctx context.Context, id string) (*domain.Product, error) {
    var model ProductModel  // struct GORM (puede ser diferente al domain)

    err := r.db.WithContext(ctx).
        Preload("Category").
        First(&model, "id = ?", id).
        Error

    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, domain.NewNotFoundError(fmt.Sprintf("producto %s no encontrado", id))
    }
    if err != nil {
        return nil, fmt.Errorf("buscar producto: %w", err)
    }

    return toDomainProduct(&model), nil
}

// Mappers — mantienen el dominio limpio
func toDomainProduct(m *ProductModel) *domain.Product {
    return &domain.Product{
        ID:          m.ID,
        Name:        m.Name,
        Price:       m.Price.InexactFloat64(),
        CategoryID:  m.CategoryID,
    }
}

func toGORMProduct(d *domain.Product) *ProductModel {
    return &ProductModel{
        Name:       d.Name,
        Price:      decimal.NewFromFloat(d.Price),
        CategoryID: d.CategoryID,
    }
}
```

---

## AutoMigrate vs Migraciones manuales

```go
// AutoMigrate — SOLO para desarrollo/prototipo
// Crea tablas y añade columnas nuevas, NUNCA elimina ni modifica columnas existentes
db.AutoMigrate(&Product{}, &Category{}, &User{})

// En producción: usar golang-migrate o goose (ver [[02-PostgreSQL-con-Go]])
// Las migraciones manuales son versionadas, reversibles y seguras
```

---

## ORM vs SQL puro — cuándo usar cada uno

| Criterio                              | ORM (GORM)           | SQL puro (pgx)        |
| ------------------------------------- | -------------------- | --------------------- |
| CRUD simple con relaciones            | ✅ Ideal             | Verbose innecesario   |
| Queries con múltiples JOINs complejos | Genera SQL subóptimo | ✅ Control total      |
| Bulk insert de miles de filas         | Lento                | ✅ `COPY FROM` de pgx |
| Aggregaciones complejas               | Limitado             | ✅                    |
| Equipo nuevo sin experiencia SQL      | ✅ Productividad     | Riesgo de errores     |
| Sistema de reporting/analytics        | No recomendable      | ✅                    |
| Prototipo rápido                      | ✅                   | Verbose               |

**Lo más común en proyectos reales**: GORM para CRUD + queries SQL puro para operaciones críticas de performance.

```go
// Escapar de GORM para una query compleja específica
var stats []ProductStats
db.Raw(`
    SELECT
        category_id,
        COUNT(*) AS total,
        AVG(price) AS avg_price
    FROM products
    WHERE deleted_at IS NULL
    GROUP BY category_id
    HAVING COUNT(*) > 5
    ORDER BY avg_price DESC
`, ).Scan(&stats)
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Dominio acoplado a GORM — viola Clean Architecture
type Product struct {
    gorm.Model  // el dominio depende de gorm — MAL
    Name  string
}

// Usar db global en lugar de inyectarlo
var DB *gorm.DB
func GetProduct(id string) *Product {
    var p Product
    DB.Find(&p, id)  // DB global — no testeable
    return &p
}
```

### Profesional

```go
// Dominio limpio
type Product struct {
    ID    string
    Name  string
    Price float64
}

// GORM solo en la capa de repositorio
// Interfaz en el dominio → implementación GORM en infra/
// Mappers para convertir entre domain.Product y gorm.ProductModel
// db inyectado por constructor → testeable con mock o DB de test
```
