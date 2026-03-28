# PostgreSQL con Go

La recomendación profesional en 2024: usa **pgx/v5** directamente. Es el driver más completo, rápido y con mejor soporte de tipos PostgreSQL.

---

## Opciones de acceso a datos

| Opción                    | Cuándo usar                             |
| ------------------------- | --------------------------------------- |
| `database/sql` + `lib/pq` | Solo si ya tenés código legado          |
| `jackc/pgx/v5`            | Nuevo proyecto, máximo control          |
| `jmoiron/sqlx` + pgx      | Quieras scan automático sin ORM         |
| `uptrace/bun`             | ORM liviano con sintaxis SQL-like       |
| GORM                      | Prototipo rápido, no producción crítica |

---

## Setup con pgx

```bash
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
```

### Pool de conexiones — siempre usar pool en producción

```go
// internal/infra/postgres/connection.go

package postgres

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

type Config struct {
    DSN             string
    MaxConnections  int32
    MinConnections  int32
    MaxConnLifetime time.Duration
    MaxConnIdleTime time.Duration
}

func NewPool(ctx context.Context, cfg Config) (*pgxpool.Pool, error) {
    poolCfg, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("parsear DSN: %w", err)
    }

    poolCfg.MaxConns        = cfg.MaxConnections  // típico: 20-50
    poolCfg.MinConns        = cfg.MinConnections  // típico: 2-5
    poolCfg.MaxConnLifetime = cfg.MaxConnLifetime // típico: 1 hora
    poolCfg.MaxConnIdleTime = cfg.MaxConnIdleTime // típico: 30 min

    pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
    if err != nil {
        return nil, fmt.Errorf("crear pool: %w", err)
    }

    if err := pool.Ping(ctx); err != nil {
        return nil, fmt.Errorf("ping a postgres: %w", err)
    }

    return pool, nil
}
```

### DSN desde variables de entorno

```go
dsn := fmt.Sprintf(
    "host=%s port=%s user=%s password=%s dbname=%s sslmode=%s",
    os.Getenv("DB_HOST"),
    os.Getenv("DB_PORT"),
    os.Getenv("DB_USER"),
    os.Getenv("DB_PASSWORD"),
    os.Getenv("DB_NAME"),
    os.Getenv("DB_SSLMODE"), // "disable" local, "require" producción
)
```

---

## Implementar el repositorio

```go
// internal/repository/postgres_product_repository.go

package repository

import (
    "context"
    "errors"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    "gestion_productos/internal/domain"
)

type postgresProductRepository struct {
    pool *pgxpool.Pool
}

func NewPostgresProductRepository(pool *pgxpool.Pool) domain.ProductRepository {
    return &postgresProductRepository{pool: pool}
}

func (r *postgresProductRepository) GetAll(ctx context.Context, filter domain.ProductFilter) ([]*domain.Product, error) {
    // Siempre usar $1, $2... NUNCA concatenar strings en SQL
    rows, err := r.pool.Query(ctx, `
        SELECT id, name, description, price, stock, category_id, created_at, updated_at
        FROM products
        WHERE deleted_at IS NULL
          AND ($1::TEXT IS NULL OR name ILIKE '%' || $1 || '%')
          AND ($2::UUID IS NULL OR category_id = $2)
        ORDER BY created_at DESC
        LIMIT $3 OFFSET $4
    `, filter.Name, filter.CategoryID, filter.Limit, filter.Offset)
    if err != nil {
        return nil, fmt.Errorf("listar productos: %w", err)
    }
    defer rows.Close()

    var products []*domain.Product
    for rows.Next() {
        p := &domain.Product{}
        err := rows.Scan(
            &p.ID, &p.Name, &p.Description, &p.Price, &p.Stock,
            &p.CategoryID, &p.CreatedAt, &p.UpdatedAt,
        )
        if err != nil {
            return nil, fmt.Errorf("escanear producto: %w", err)
        }
        products = append(products, p)
    }

    // CRÍTICO: verificar error de iteración
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterar productos: %w", err)
    }

    return products, nil
}

func (r *postgresProductRepository) GetByID(ctx context.Context, id string) (*domain.Product, error) {
    p := &domain.Product{}
    err := r.pool.QueryRow(ctx, `
        SELECT id, name, description, price, stock, category_id, created_at, updated_at
        FROM products
        WHERE id = $1 AND deleted_at IS NULL
    `, id).Scan(
        &p.ID, &p.Name, &p.Description, &p.Price, &p.Stock,
        &p.CategoryID, &p.CreatedAt, &p.UpdatedAt,
    )

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            // Traducir error de DB → error de dominio
            return nil, domain.NewNotFoundError(fmt.Sprintf("producto %s no encontrado", id))
        }
        return nil, fmt.Errorf("obtener producto %s: %w", id, err)
    }

    return p, nil
}

func (r *postgresProductRepository) Create(ctx context.Context, p *domain.Product) (*domain.Product, error) {
    err := r.pool.QueryRow(ctx, `
        INSERT INTO products (name, description, price, stock, category_id)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id, created_at, updated_at
    `, p.Name, p.Description, p.Price, p.Stock, p.CategoryID).
        Scan(&p.ID, &p.CreatedAt, &p.UpdatedAt)

    if err != nil {
        // Detectar violación de constraint
        if isUniqueViolation(err) {
            return nil, domain.NewConflictError("ya existe un producto con ese nombre")
        }
        return nil, fmt.Errorf("crear producto: %w", err)
    }

    return p, nil
}

// Detectar errores específicos de PostgreSQL
func isUniqueViolation(err error) bool {
    var pgErr *pgconn.PgError
    return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

---

## Transacciones en Go con pgx

```go
func (r *postgresProductRepository) CreateWithInventoryAdjustment(
    ctx context.Context,
    product *domain.Product,
    adjustment domain.InventoryAdjustment,
) (*domain.Product, error) {
    // Iniciar transacción
    tx, err := r.pool.Begin(ctx)
    if err != nil {
        return nil, fmt.Errorf("iniciar transacción: %w", err)
    }
    // defer rollback — si Commit() no fue llamado, se hace rollback
    // pgx hace rollback automático si la transacción no fue committed
    defer tx.Rollback(ctx)

    // Insertar producto
    err = tx.QueryRow(ctx, `
        INSERT INTO products (name, price, stock, category_id)
        VALUES ($1, $2, $3, $4)
        RETURNING id, created_at
    `, product.Name, product.Price, product.Stock, product.CategoryID).
        Scan(&product.ID, &product.CreatedAt)
    if err != nil {
        return nil, fmt.Errorf("crear producto en tx: %w", err)
    }

    // Registrar ajuste de inventario
    _, err = tx.Exec(ctx, `
        INSERT INTO inventory_adjustments (product_id, quantity, reason, created_at)
        VALUES ($1, $2, $3, NOW())
    `, product.ID, adjustment.Quantity, adjustment.Reason)
    if err != nil {
        return nil, fmt.Errorf("crear ajuste de inventario en tx: %w", err)
    }

    // Commit — confirma todo
    if err := tx.Commit(ctx); err != nil {
        return nil, fmt.Errorf("confirmar transacción: %w", err)
    }

    return product, nil
}
```

---

## Migraciones con golang-migrate

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

### Estructura de migraciones

```
migrations/
├── 000001_create_categories.up.sql
├── 000001_create_categories.down.sql
├── 000002_create_users.up.sql
├── 000002_create_users.down.sql
├── 000003_create_products.up.sql
└── 000003_create_products.down.sql
```

```sql
-- 000003_create_products.up.sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    price       DECIMAL(12,2) NOT NULL CHECK (price >= 0),
    stock       INTEGER NOT NULL DEFAULT 0,
    category_id UUID NOT NULL REFERENCES categories(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_name        ON products(name);

-- 000003_create_products.down.sql
DROP TABLE IF EXISTS products;
```

### Ejecutar migraciones

```bash
# Aplicar todas las migraciones pendientes
migrate -path ./migrations -database "postgres://user:pass@localhost:5432/dbname?sslmode=disable" up

# Revertir la última migración
migrate -path ./migrations -database "..." down 1

# Ver versión actual
migrate -path ./migrations -database "..." version
```

### Migraciones desde código (al iniciar la app)

```go
import "github.com/golang-migrate/migrate/v4"
import _ "github.com/golang-migrate/migrate/v4/database/postgres"
import _ "github.com/golang-migrate/migrate/v4/source/file"

func RunMigrations(dsn string) error {
    m, err := migrate.New("file://migrations", dsn)
    if err != nil {
        return fmt.Errorf("crear migración: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return fmt.Errorf("ejecutar migraciones: %w", err)
    }

    return nil
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// SQL concatenado → SQL injection vulnerable
query := "SELECT * FROM products WHERE name = '" + name + "'"

// Sin pool — nueva conexión por request
db, _ := sql.Open("postgres", dsn) // MAL: abrir en cada función
defer db.Close()
```

### Profesional

```go
// 1. Queries parametrizadas — siempre $1, $2
row := pool.QueryRow(ctx, "SELECT * FROM products WHERE name = $1", name)

// 2. Pool compartido — crear una vez en main.go, pasar por DI
// 3. Errores traducidos — pgx.ErrNoRows → domain.NotFoundError
// 4. Context en todas las queries — permite cancelación y timeouts

// 5. Timeout por operación
queryCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
rows, err := pool.Query(queryCtx, "SELECT ...")

// 6. Verificar rows.Err() siempre
defer rows.Close()
for rows.Next() { ... }
if err := rows.Err(); err != nil {
    return nil, fmt.Errorf("iterar: %w", err)
}
```
