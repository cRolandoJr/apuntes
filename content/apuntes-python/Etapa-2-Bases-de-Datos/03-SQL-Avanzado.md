# SQL Avanzado — Más allá del ORM

La guía de Go tiene `01-SQL-Fundamentos.md`. Este archivo cubre los temas de SQL que necesitás manejar cuando el ORM no alcanza, o cuando querés entender qué está generando.

---

## JOINs — unir tablas

```sql
-- INNER JOIN: solo filas que tienen coincidencia en AMBAS tablas
SELECT p.id, p.name, c.name AS category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.id;

-- LEFT JOIN: todas las filas de la izquierda, con o sin coincidencia
-- Las que no coinciden traen NULL en las columnas de la tabla derecha
SELECT u.id, u.email, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
-- Usuarios sin pedidos: aparecen con order_id = NULL

-- RIGHT JOIN: lo opuesto al LEFT (raro de ver, casi siempre se puede reescribir como LEFT)
SELECT o.id, u.email
FROM orders o
RIGHT JOIN users u ON o.user_id = u.id;
-- Equivalente al LEFT JOIN de arriba con tablas invertidas

-- FULL OUTER JOIN: todas las filas de ambas tablas, con o sin coincidencia
SELECT p.name AS product, o.id AS order_id
FROM products p
FULL OUTER JOIN order_items oi ON oi.product_id = p.id
FULL OUTER JOIN orders o ON o.id = oi.order_id;
```

### Regla práctica

| Querés ver                               | JOIN a usar                    |
| ---------------------------------------- | ------------------------------ |
| Solo los que tienen relación             | INNER JOIN                     |
| Todos de la tabla A (vengan o no de B)   | LEFT JOIN                      |
| Reportes — incluir también los huérfanos | LEFT JOIN                      |
| Buscar huérfanos                         | LEFT JOIN + WHERE B.id IS NULL |

---

## Subqueries

```sql
-- Subquery en WHERE
SELECT name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
-- "productos más caros que el promedio"

-- Subquery en FROM (derived table)
SELECT dept, avg_salary
FROM (
    SELECT department AS dept, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) AS dept_stats
WHERE avg_salary > 50000;

-- Subquery correlacionada — se ejecuta una vez por cada fila del outer query
-- Lenta en tablas grandes, usar con cuidado
SELECT p.name
FROM products p
WHERE EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);
-- "productos que han sido pedidos al menos una vez"

-- Lo mismo más eficiente con JOIN:
SELECT DISTINCT p.name
FROM products p
INNER JOIN order_items oi ON oi.product_id = p.id;
```

### Cuándo usar subquery vs JOIN

- **JOIN** cuando necesitás columnas de ambas tablas
- **Subquery** cuando solo necesitás filtrar basado en otra tabla
- **EXISTS** es más eficiente que `IN (SELECT ...)` para subqueries grandes

---

## Window Functions — cálculos sobre grupos sin colapsar filas

Las window functions hacen cálculos sobre un conjunto de filas relacionadas pero **sin** reducir el resultado a una sola fila (a diferencia de GROUP BY).

```sql
-- ROW_NUMBER — numeración secuencial por grupo
SELECT
    id,
    name,
    category_id,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS rank_in_category
FROM products;
-- Cada producto numerado dentro de su categoría, ordenado de más caro a más barato

-- RANK — igual que ROW_NUMBER pero con empates (puede saltar números: 1, 2, 2, 4)
SELECT name, price, RANK() OVER (ORDER BY price DESC) AS price_rank
FROM products;

-- LAG / LEAD — acceder a la fila anterior/siguiente
SELECT
    date,
    revenue,
    LAG(revenue) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1, 0) OVER (ORDER BY date) AS daily_change
FROM daily_sales;

-- SUM acumulativo
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue
FROM daily_sales;

-- Estadísticas por grupo sin perder detalle de filas
SELECT
    id,
    name,
    price,
    AVG(price) OVER (PARTITION BY category_id) AS category_avg_price,
    price - AVG(price) OVER (PARTITION BY category_id) AS diff_from_avg
FROM products;
```

### Anatomía de OVER

```sql
FUNCION() OVER (
    PARTITION BY columna_de_agrupamiento   -- opcional — como GROUP BY pero sin colapsar
    ORDER BY columna_de_orden              -- obligatorio para ROW_NUMBER, LAG, SUM acumulativo
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- opcional — ventana de filas
)
```

---

## EXPLAIN ANALYZE — entender qué hace la DB

```sql
-- EXPLAIN: muestra el plan sin ejecutar la query
EXPLAIN SELECT * FROM products WHERE category_id = 5;

-- EXPLAIN ANALYZE: ejecuta la query Y muestra el plan con tiempos reales
EXPLAIN ANALYZE SELECT * FROM products WHERE category_id = 5;

-- EXPLAIN (ANALYZE, BUFFERS): incluye info de uso de cache/disco
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.name, c.name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.price > 100;
```

### Leer el output de EXPLAIN

```
Seq Scan on products  (cost=0.00..45.00 rows=1000 width=64) (actual time=0.01..0.8 rows=1000 loops=1)
  Filter: (price > 100)
  Rows Removed by Filter: 150
```

| Término             | Qué significa                                                            |
| ------------------- | ------------------------------------------------------------------------ |
| `Seq Scan`          | Recorrió toda la tabla — puede ser problema en tablas grandes            |
| `Index Scan`        | Usó un índice — generalmente bueno                                       |
| `Bitmap Index Scan` | Usó índice + bitmap — común para múltiples condiciones                   |
| `Hash Join`         | Join usando hash table — eficiente para tablas medianas                  |
| `Nested Loop`       | Join de loops anidados — eficiente si la tabla interna es pequeña        |
| `cost=X..Y`         | Costo estimado (X=startup, Y=total). Son unidades relativas, no segundos |
| `actual time=X..Y`  | Tiempo real en ms (solo con ANALYZE)                                     |
| `rows=N`            | Filas estimadas / reales                                                 |

**Señales de alerta:**

- `Seq Scan` en tablas con millones de filas con un WHERE
- Grandes discrepancias entre `rows` estimadas y reales (estadísticas desactualizadas → `ANALYZE tabla;`)
- `cost` muy alto o nested loops en tablas grandes

---

## Índices

```sql
-- B-tree (default) — para comparaciones: =, <, >, BETWEEN, LIKE 'abc%'
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);

-- Índice compuesto — útil cuando siempre filtrás por las dos columnas juntas
-- Orden importa: la primera columna se puede usar sola, la segunda requiere la primera
CREATE INDEX idx_products_cat_price ON products(category_id, price);

-- Índice parcial — solo indexa un subconjunto (más pequeño y rápido)
CREATE INDEX idx_active_products ON products(category_id) WHERE active = true;
-- Útil cuando el 90% de las queries solo buscan productos activos

-- GIN — full text search, arrays, JSONB
CREATE INDEX idx_products_tags ON products USING GIN(tags);   -- tags es un array
-- Para búsquedas: SELECT * FROM products WHERE tags @> ARRAY['electronics'];

-- Índice de expresión
CREATE INDEX idx_products_lower_name ON products(LOWER(name));
-- Para: SELECT * FROM products WHERE LOWER(name) = 'camisa';

-- Ver índices de una tabla
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'products';

-- Ver qué índices se están usando (cuánto se usan)
SELECT relname, indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE relname = 'products';
-- idx_scan = 0 → el índice nunca se usa → candidato para borrar
```

### Cuándo un índice NO ayuda

- Tablas muy pequeñas (PostgreSQL prefiere Seq Scan)
- Columnas con muy poca cardinalidad (ej: columna `activo` con solo true/false)
- Cuando la query devuelve más del ~15% de las filas (escanear el índice + las filas cuesta más que Seq Scan)

---

## SQL crudo con SQLAlchemy

A veces el ORM genera queries ineficientes o necesitás funcionalidad que no soporta. Podés usar SQL crudo manteniendo la protección contra SQL injection:

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

async def get_products_with_stats(
    session: AsyncSession,
    category_id: int,
) -> list[dict]:
    """Query con window function — no se puede expresar fácilmente con ORM."""

    stmt = text("""
        SELECT
            p.id,
            p.name,
            p.price,
            AVG(p.price) OVER (PARTITION BY p.category_id)::NUMERIC(10,2) AS category_avg,
            RANK() OVER (PARTITION BY p.category_id ORDER BY p.price DESC) AS price_rank
        FROM products p
        WHERE p.category_id = :category_id
          AND p.active = true
        ORDER BY price_rank
    """)

    # :category_id — parámetro nombrado, SQLAlchemy lo escapa automáticamente
    result = await session.execute(stmt, {"category_id": category_id})

    # Convertir a lista de dicts
    return [row._asdict() for row in result.fetchall()]


async def bulk_update_prices(
    session: AsyncSession,
    category_id: int,
    factor: float,
) -> int:
    """UPDATE masivo más eficiente que actualizar fila por fila."""

    stmt = text("""
        UPDATE products
        SET price = price * :factor,
            updated_at = NOW()
        WHERE category_id = :category_id
        RETURNING id
    """)

    result = await session.execute(stmt, {
        "factor": factor,
        "category_id": category_id,
    })
    await session.commit()

    updated_ids = result.fetchall()
    return len(updated_ids)
```

---

## N+1 queries — el problema más común con ORMs

```python
# MAL — N+1 queries: 1 query para los productos + 1 por cada producto para la categoría
products = await session.execute(select(Product))
for product in products.scalars():
    print(product.category.name)   # query extra por cada producto

# BIEN — eager loading: 1 sola query con JOIN
from sqlalchemy.orm import selectinload, joinedload

# selectinload: hace 2 queries (1 para productos, 1 para todas las categorías)
# Mejor para colecciones (evita duplicar filas)
stmt = select(Product).options(selectinload(Product.category))
products = await session.execute(stmt)

# joinedload: hace 1 query con JOIN
# Mejor para relaciones many-to-one (category de un product)
stmt = select(Product).options(joinedload(Product.category))
products = await session.execute(stmt)
```

### Detectar N+1 en desarrollo

```python
# settings.py — solo en desarrollo/test
# Ver cada query SQL en la consola
SQLALCHEMY_ECHO = True   # en la configuración de create_async_engine

# O contar queries en tests
from sqlalchemy import event

query_count = 0

@event.listens_for(engine.sync_engine, "before_cursor_execute")
def count_queries(conn, cursor, statement, parameters, context, executemany):
    global query_count
    query_count += 1

# En el test
query_count = 0
result = await service.get_products_with_categories()
assert query_count <= 2, f"Demasiadas queries: {query_count}"
```

---

## Transacciones

```python
# SQLAlchemy maneja la transacción automáticamente por sesión
# session.commit() confirma, session.rollback() revierte

# Savepoints — para rollback parcial dentro de una transacción
async def transfer_stock(session: AsyncSession, from_id: int, to_id: int, qty: int):
    async with session.begin_nested() as savepoint:   # SAVEPOINT
        try:
            await decrease_stock(session, from_id, qty)
            await increase_stock(session, to_id, qty)
        except InsufficientStockError:
            await savepoint.rollback()   # solo revierte este bloque
            raise
    # El commit del bloque padre confirma todo
```
