# SQL Fundamentos para Backend

SQL es el lenguaje de las bases de datos relacionales. Todo backend trabaja con datos persistentes — esta es la habilidad más valiosa después del lenguaje principal.

---

## Modelo de datos — diseña primero en papel

```sql
-- Regla: normalización evita datos inconsistentes

CREATE TABLE categories (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    role            VARCHAR(20) NOT NULL DEFAULT 'user',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    price       DECIMAL(12, 2) NOT NULL CHECK (price >= 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
    created_by  UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Índices — sin índice = full table scan en producción
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_name        ON products(name);
CREATE INDEX idx_products_price       ON products(price);
-- Índice compuesto — útil para "filtrar por categoría ordenado por nombre"
CREATE INDEX idx_products_category_name ON products(category_id, name);
```

---

## SELECT — lectura de datos

```sql
-- Básico
SELECT id, name, price FROM products WHERE stock > 0;

-- Todas las columnas (evitar en producción — incluye columnas que no necesitás)
SELECT * FROM products;

-- Filtros
SELECT * FROM products
WHERE price BETWEEN 100 AND 500
  AND category_id = '550e8400-e29b-41d4-a716-446655440000'
  AND name ILIKE '%laptop%';  -- ILIKE = case-insensitive

-- NULL especial — no usar = NULL, usar IS NULL
SELECT * FROM products WHERE created_by IS NULL;
SELECT * FROM products WHERE created_by IS NOT NULL;

-- Ordenamiento y paginación
SELECT id, name, price, stock
FROM products
ORDER BY price DESC, name ASC
LIMIT 20 OFFSET 40;  -- página 3 (0-indexed: offset = (page-1)*pageSize)

-- Agregaciones
SELECT
    category_id,
    COUNT(*)          AS total_products,
    AVG(price)        AS avg_price,
    MIN(price)        AS min_price,
    MAX(price)        AS max_price,
    SUM(stock)        AS total_stock
FROM products
GROUP BY category_id
HAVING COUNT(*) > 5   -- HAVING filtra después de agrupar (WHERE filtra antes)
ORDER BY total_products DESC;

-- Distinct
SELECT DISTINCT category_id FROM products;
```

---

## JOINs — combinar tablas

```sql
-- INNER JOIN — solo filas con coincidencia en AMBAS tablas
SELECT
    p.id,
    p.name,
    p.price,
    c.name AS category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.id;

-- LEFT JOIN — todas las filas de la izquierda, NULL si no hay match
SELECT
    u.email,
    COUNT(p.id) AS products_created
FROM users u
LEFT JOIN products p ON p.created_by = u.id
GROUP BY u.id, u.email;

-- Múltiples JOINs
SELECT
    p.name,
    c.name  AS category,
    u.email AS created_by_email
FROM products p
INNER JOIN categories c ON p.category_id = c.id
LEFT JOIN  users u     ON p.created_by = u.id
WHERE p.stock > 0
ORDER BY p.created_at DESC
LIMIT 20;
```

---

## INSERT, UPDATE, DELETE

```sql
-- INSERT
INSERT INTO products (name, description, price, stock, category_id)
VALUES ('Laptop Pro', 'Laptop profesional', 1500.00, 10, '550e...')
RETURNING id, created_at;  -- RETURNING devuelve el registro insertado

-- INSERT múltiple
INSERT INTO products (name, price, stock, category_id)
VALUES
    ('Producto A', 100.00, 5, 'cat-id-1'),
    ('Producto B', 200.00, 3, 'cat-id-2');

-- UPDATE
UPDATE products
SET
    price = 1400.00,
    stock = stock - 1,  -- operación atómica
    updated_at = NOW()
WHERE id = 'product-id'
RETURNING id, price, stock;

-- DELETE
DELETE FROM products WHERE id = 'product-id';

-- Soft delete — preferible en producción
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;
UPDATE products SET deleted_at = NOW() WHERE id = 'product-id';
SELECT * FROM products WHERE deleted_at IS NULL;  -- siempre filtrar
```

---

## Transacciones — ACID

```sql
-- ACID: Atomic, Consistent, Isolated, Durable

BEGIN;

-- Todas estas operaciones forman una unidad
UPDATE products SET stock = stock - 1 WHERE id = 'product-id' AND stock > 0;

-- Si el UPDATE no afectó ninguna fila, algo salió mal
-- El código Go verifica rows_affected

INSERT INTO orders (product_id, user_id, quantity, total_price)
VALUES ('product-id', 'user-id', 1, 1500.00);

COMMIT;   -- confirmar todo
-- ROLLBACK;  -- deshacer todo si algo falló
```

### Isolation levels

```sql
-- Por defecto: READ COMMITTED (suficiente para la mayoría de casos)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Para operaciones críticas (inventario, pagos): SERIALIZABLE
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## Subqueries y CTEs

```sql
-- CTE (Common Table Expression) — más legible que subqueries anidadas
WITH expensive_products AS (
    SELECT * FROM products WHERE price > 1000
),
products_with_category AS (
    SELECT
        ep.*,
        c.name AS category_name
    FROM expensive_products ep
    INNER JOIN categories c ON ep.category_id = c.id
)
SELECT * FROM products_with_category ORDER BY price DESC;

-- EXISTS — más eficiente que IN para subqueries grandes
SELECT * FROM categories c
WHERE EXISTS (
    SELECT 1 FROM products p
    WHERE p.category_id = c.id AND p.stock > 0
);
```

---

## EXPLAIN ANALYZE — diagnosticar queries lentas

```sql
-- Analiza el plan de ejecución real
EXPLAIN ANALYZE
SELECT p.name, c.name
FROM products p
INNER JOIN categories c ON p.category_id = c.id
WHERE p.price > 500;

-- Cosas a buscar:
-- Seq Scan → full table scan (peligroso en tablas grandes)
-- Index Scan → usa índice (bueno)
-- Hash Join / Merge Join → OK para JOINs
-- Sort → puede requerir memoria (crear índice para ordenamiento frecuente)
```

---

## Patrones importantes

```sql
-- Upsert — insert o update si ya existe
INSERT INTO products (id, name, price)
VALUES ('id', 'Producto', 100.00)
ON CONFLICT (id) DO UPDATE
SET name = EXCLUDED.name, price = EXCLUDED.price, updated_at = NOW();

-- Window functions — cálculos por grupos sin colapsar filas
SELECT
    name,
    price,
    category_id,
    RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS price_rank
FROM products;

-- Buscar top-3 más caros por categoría
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS rn
    FROM products
) t WHERE t.rn <= 3;
```

---

## Reglas de oro

| Práctica                            | Por qué                        |
| ----------------------------------- | ------------------------------ |
| Usar UUID o BIGSERIAL para IDs      | Escalabilidad, no filtran info |
| Siempre TIMESTAMPTZ (con zona)      | Evita bugs de timezone         |
| Índice en cada FK                   | JOINs rápidos                  |
| CHECK constraints en el schema      | Integridad de datos en la BD   |
| Soft delete                         | Datos recuperables, auditoría  |
| LIMIT en todas las queries de lista | Evitar traer 10M filas         |
| Queries parametrizadas              | Prevenir SQL injection         |
| RETURNING en INSERT/UPDATE          | Evitar un SELECT extra         |
