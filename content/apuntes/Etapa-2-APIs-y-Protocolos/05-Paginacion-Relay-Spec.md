# Paginación — Relay Cursor Spec y Patrones

La paginación es uno de los temas que más diferencia una API amateur de una profesional. La forma en que paginás determina la consistencia de datos, la performance y la experiencia del cliente.

> Este archivo complementa directamente `[[03-GraphQL-La-Especificacion]]` y `[[03-ORM-y-Transacciones]]`.

---

## Los dos enfoques principales

### Offset-based (basada en offset)

```sql
SELECT * FROM products ORDER BY created_at DESC LIMIT 20 OFFSET 40;
-- Página 3 de resultados de 20 items (0-indexed: offset = (3-1)*20)
```

```graphql
query {
  products(page: 3, pageSize: 20) {
    id
    name
  }
}
```

**Problemas:**

```
Inserción mientras paginás:

Página 1 (t=0): items [1,2,3,4,5] → usuario ve los 5
Nuevo item insertado al inicio (t=1)
Página 2 (t=2): items ahora son [nuevo,1,2,3,4,5,6...]
  → OFFSET 5: retorna [5,6,7,8,9] → ¡item 5 aparece DOS VECES!
  → item 6 nunca aparece en ninguna página ("phantom item")
```

Además: `OFFSET 10000` en PostgreSQL lee y descarta 10.000 filas — degrada en O(n).

**Cuándo usarlo**: paginación de backoffice, dashboards admin, resultados de búsqueda donde no importa consistencia perfecta y el dataset es pequeño.

### Cursor-based (basada en cursor) — el estándar de producción

```
En lugar de "dame la página 3", usas "dame los items DESPUÉS del cursor X"

Cursor = posición estable en el resultado (no cambia si se insertan items)
```

```sql
-- Cursor basado en created_at + id (evita duplicados si hay timestamps iguales)
SELECT * FROM products
WHERE (created_at, id) < ($1, $2)  -- menor que el cursor del último item
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Ventajas:

- Consistente aunque se inserten/eliminen items entre requests
- Performance constante O(log n) con el índice correcto
- No tiene el problema de "items duplicados o saltados"

---

## Relay Cursor Specification — el estándar de GraphQL

Relay (el cliente GraphQL de Facebook/Meta) definió una spec de paginación que se volvió el estándar de facto para APIs GraphQL. Tiene dos partes: **Connections** y **Global Object Identification**.

### Connections — la estructura

```graphql
# En lugar de devolver [Product!]! directamente:
type Query {
  products(
    first: Int # los primeros N items (hacia adelante)
    after: String # cursor del item después del cual empezar
    last: Int # los últimos N items (hacia atrás)
    before: String # cursor del item antes del cual empezar
    filter: ProductFilter
  ): ProductConnection!
}

# La Connection
type ProductConnection {
  edges: [ProductEdge] # los items con su cursor
  pageInfo: PageInfo! # metadatos de paginación
  totalCount: Int! # total de items (para "mostrando 1-20 de 150")
}

# El Edge — envuelve cada item con su cursor
type ProductEdge {
  node: Product! # el item real
  cursor: String! # posición de este item para paginación
}

# PageInfo — info para el cliente
type PageInfo {
  hasNextPage: Boolean! # ¿hay más items hacia adelante?
  hasPreviousPage: Boolean! # ¿hay más items hacia atrás?
  startCursor: String # cursor del primer item en este resultado
  endCursor: String # cursor del último item en este resultado
}
```

### Usar la Connection en una query

```graphql
# Avanzar hacia adelante (forward pagination)
query ListProducts($after: String) {
  products(first: 20, after: $after) {
    totalCount
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      cursor
      node {
        id
        name
        price
      }
    }
  }
}
```

Variables primera carga:

```json
{ "after": null }
```

Variables siguiente página (usar `endCursor` de la respuesta anterior):

```json
{ "after": "eyJpZCI6IjEyMyIsInRzIjoiMjAyNC0wMy0yOFQxNDozMDowMFoifQ==" }
```

---

## Implementar Relay Connections en Go (gqlgen)

### Schema

```graphql
# En schema.graphqls

type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ProductEdge {
  node: Product!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  products(
    first: Int
    after: String
    last: Int
    before: String
    filter: ProductFilter
  ): ProductConnection!
}
```

### El cursor — codificado en base64

```go
// El cursor es opaco para el cliente — el cliente no debe parsearlo
// Internamente es: base64(json({id: "abc", ts: "2024-03-28T14:30:00Z"}))

package cursor

import (
    "encoding/base64"
    "encoding/json"
    "fmt"
    "time"
)

type ProductCursor struct {
    CreatedAt time.Time `json:"ts"`
    ID        string    `json:"id"`
}

func Encode(id string, createdAt time.Time) string {
    c := ProductCursor{ID: id, CreatedAt: createdAt}
    b, _ := json.Marshal(c)
    return base64.StdEncoding.EncodeToString(b)
}

func Decode(encoded string) (*ProductCursor, error) {
    b, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil {
        return nil, fmt.Errorf("cursor inválido: %w", err)
    }
    var c ProductCursor
    if err := json.Unmarshal(b, &c); err != nil {
        return nil, fmt.Errorf("cursor malformado: %w", err)
    }
    return &c, nil
}
```

### Repositorio — la query de cursor

```go
type CursorPage struct {
    First  *int    // primeros N items
    After  *string // cursor de inicio
    Last   *int    // últimos N items (paginación inversa)
    Before *string // cursor de fin
}

func (r *postgresProductRepo) GetPage(ctx context.Context, filter ProductFilter, page CursorPage) (*ProductConnection, error) {
    limit := 20
    if page.First != nil {
        limit = *page.First
    }

    query := `
        SELECT id, name, price, stock, created_at
        FROM products
        WHERE deleted_at IS NULL
    `
    args := []interface{}{}
    argN := 1

    // Cursor condition
    if page.After != nil {
        c, err := cursor.Decode(*page.After)
        if err != nil {
            return nil, err
        }
        query += fmt.Sprintf(" AND (created_at, id) < ($%d, $%d)", argN, argN+1)
        args = append(args, c.CreatedAt, c.ID)
        argN += 2
    }

    query += fmt.Sprintf(" ORDER BY created_at DESC, id DESC LIMIT $%d + 1", argN)
    args = append(args, limit)
    // Pedimos LIMIT+1 para saber si hay más páginas

    rows, err := r.pool.Query(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var products []*Product
    for rows.Next() {
        p := &Product{}
        rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock, &p.CreatedAt)
        products = append(products, p)
    }

    // Detectar si hay página siguiente
    hasNextPage := len(products) > limit
    if hasNextPage {
        products = products[:limit] // quitar el item extra
    }

    // Construir edges
    edges := make([]*ProductEdge, len(products))
    for i, p := range products {
        edges[i] = &ProductEdge{
            Node:   p,
            Cursor: cursor.Encode(p.ID, p.CreatedAt),
        }
    }

    // PageInfo
    pageInfo := &PageInfo{
        HasNextPage:     hasNextPage,
        HasPreviousPage: page.After != nil,
    }
    if len(edges) > 0 {
        pageInfo.StartCursor = &edges[0].Cursor
        pageInfo.EndCursor   = &edges[len(edges)-1].Cursor
    }

    // Total count (query separada, puede ser cacheada)
    var totalCount int64
    r.pool.QueryRow(ctx, "SELECT COUNT(*) FROM products WHERE deleted_at IS NULL").Scan(&totalCount)

    return &ProductConnection{
        Edges:      edges,
        PageInfo:   pageInfo,
        TotalCount: int(totalCount),
    }, nil
}
```

---

## Global Object Identification — la otra parte de Relay

Relay requiere que cada objeto tenga un ID global único y que exista un campo `node(id: ID!): Node` en el schema.

```graphql
# Interfaz Node
interface Node {
  id: ID!
}

type Product implements Node {
  id: ID!
  name: String!
  # ...
}

type User implements Node {
  id: ID!
  email: String!
  # ...
}

type Query {
  # Buscar cualquier entidad por ID global
  node(id: ID!): Node
  nodes(ids: [ID!]!): [Node]! # batch
}
```

El ID global codifica el tipo + el ID de base de datos:

```go
// "Product:abc-123" → base64 → "UHJvZHVjdDphYmMtMTIz"
func EncodeGlobalID(typeName, id string) string {
    return base64.StdEncoding.EncodeToString([]byte(typeName + ":" + id))
}

func DecodeGlobalID(globalID string) (typeName, id string, err error) {
    decoded, err := base64.StdEncoding.DecodeString(globalID)
    if err != nil {
        return "", "", err
    }
    parts := strings.SplitN(string(decoded), ":", 2)
    if len(parts) != 2 {
        return "", "", errors.New("global ID inválido")
    }
    return parts[0], parts[1], nil
}
```

```go
// Resolver del campo node en gqlgen
func (r *queryResolver) Node(ctx context.Context, id string) (model.Node, error) {
    typeName, dbID, err := cursor.DecodeGlobalID(id)
    if err != nil {
        return nil, domain.NewInvalidInputError("ID inválido")
    }

    switch typeName {
    case "Product":
        return r.ProductUC.GetByID(ctx, dbID)
    case "User":
        return r.UserUC.GetByID(ctx, dbID)
    default:
        return nil, domain.NewNotFoundError("tipo desconocido")
    }
}
```

---

## Comparación de patrones

| Patrón               | Consistencia               | Performance en pág. alta | Backend       |
| -------------------- | -------------------------- | ------------------------ | ------------- |
| Offset               | Baja (duplicados posibles) | O(offset) — degrada      | Muy simple    |
| Keyset/Cursor        | Alta                       | O(log n) con índice      | Moderado      |
| Relay ConnectionSpec | Alta                       | O(log n)                 | Moderado-alto |

**Recomendación práctica:**

- API pública / móvil → Relay Connection Spec (cursor-based)
- Panel admin / reporting → offset (acceptable para datasets pequeños)
- Feeds / timelines en tiempo real → cursor-based siempre

---

## Índices necesarios para cursor pagination eficiente

```sql
-- Para cursor (created_at DESC, id DESC)
CREATE INDEX idx_products_cursor
ON products(created_at DESC, id DESC)
WHERE deleted_at IS NULL;  -- partial index — excluye soft-deleted
```

Sin este índice, PostgreSQL hace un full scan y sort en cada request de paginación.
