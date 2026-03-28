# GraphQL — La Especificación Completa

GraphQL no es una librería ni un framework. Es un **lenguaje de query y un protocolo** definido por la especificación oficial (spec.graphql.org). Cualquier servidor que implemente la spec es un servidor GraphQL válido, sin importar el lenguaje.

> Relación con gqlgen: `[[03-GraphQL]]` cubre la implementación en Go. Este archivo cubre la base teórica que aplica a cualquier lenguaje.

---

## ¿Qué problema resuelve?

### REST — los problemas clásicos

```
Overfetching: pedir /users/42 y recibir 30 campos cuando solo necesitabas nombre y email

Underfetching: mostrar un perfil requiere:
  GET /users/42
  GET /users/42/posts
  GET /users/42/followers
  → 3 round trips para una sola pantalla

Versionado: /api/v1/... → /api/v2/... → deprecaciones, código muerto, endpoints infinitos
```

### GraphQL — un solo endpoint, el cliente decide

```graphql
# En una sola request, el cliente pide exactamente lo que necesita
query PerfilCompleto($userId: ID!) {
  user(id: $userId) {
    name
    email
    posts(last: 5) {
      title
      publishedAt
    }
    followersCount
  }
}
```

Un solo endpoint (típicamente `POST /graphql`), sin versiones, sin overfetching.

---

## El Sistema de Tipos — Type System

El schema es el contrato entre el cliente y el servidor. Todo lo que puede pedirse o enviarse debe estar en el schema.

### Tipos escalares (Scalar Types)

Son los tipos hoja — no tienen sub-campos.

```graphql
# 5 escalares built-in:
Int     # entero con signo de 32 bits: -2147483648 a 2147483647
Float   # número de punto flotante de doble precisión (IEEE 754)
String  # secuencia UTF-8
Boolean # true o false
ID      # identificador único — se serializa como String, pero semánticamente es un ID

# Escalares personalizados — el servidor define cómo serializarlos/parsearlos
scalar DateTime   # típicamente: ISO 8601 → "2024-03-28T14:30:00Z"
scalar Decimal    # monetario, para evitar imprecisión de Float
scalar URL        # para validar formato
scalar UUID       # UUID v4
scalar JSON       # objeto JSON sin tipado (útil pero peligroso — pierde type safety)
```

### Object Types — el corazón del schema

```graphql
type Product {
  id: ID! # ! = Non-Null: el servidor GARANTIZA que nunca será null
  name: String!
  price: Float!
  description: String # sin ! = Nullable: puede ser null
  category: Category # campo que resuelve otro Object Type
  tags: [String!]! # lista non-null de strings non-null
  relatedProducts: [Product!] # nullable list (la lista puede ser null)
}

type Category {
  id: ID!
  name: String!
  products: [Product!]!
}
```

### Los 3 tipos raíz — Query, Mutation, Subscription

```graphql
# Son Object Types especiales — son los puntos de entrada al schema

type Query {
  # Operaciones de lectura, resueltas en PARALELO
  product(id: ID!): Product
  products(filter: ProductFilter, page: Int, pageSize: Int): [Product!]!
  me: User
}

type Mutation {
  # Operaciones de escritura, resueltas en SERIE (una después de otra)
  createProduct(input: CreateProductInput!): Product!
  updateProduct(id: ID!, input: UpdateProductInput!): Product!
  deleteProduct(id: ID!): DeleteResult!
}

type Subscription {
  # Eventos en tiempo real
  # RESTRICCIÓN SPEC: EXACTAMENTE 1 campo raíz
  productUpdated(id: ID!): Product!
  newProductCreated: Product!
}
```

### Input Types — solo para argumentos

```graphql
# Los Input Types son tipos SEPARADOS de los Object Types
# No se pueden mezclar: un Input type NO puede tener campos que resuelvan Object types

input CreateProductInput {
  name: String!
  price: Float!
  stock: Int!
  categoryId: ID!
  # Los campos de inputs también pueden ser non-null o nullable
  description: String # opcional
  tags: [String!] # lista opcional
}

input ProductFilter {
  name: String
  minPrice: Float
  maxPrice: Float
  inStock: Boolean
}

# ¿Por qué separar Input de Output?
# Un Product tiene un campo category: Category (tipo de salida)
# Un CreateProductInput tiene categoryId: ID — no puede tener category: Category
# porque Category es un Output type que requiere resolvers
```

### Enums

```graphql
enum ProductStatus {
  DRAFT
  ACTIVE
  DISCONTINUED
}

enum OrderDirection {
  ASC
  DESC
}

type Query {
  products(orderBy: String, direction: OrderDirection = ASC): [Product!]!
  #                                                   ^^^^^ valor default
}
```

### Interfaces — comportamiento compartido

```graphql
interface Node {
  id: ID! # Relay spec: todo lo que tiene ID implementa Node
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Product implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  price: Float!
}

type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  email: String!
}

# Usar la interfaz en queries
type Query {
  node(id: ID!): Node # retorna cualquier tipo que implemente Node
}
```

### Unions — varios tipos posibles sin campos comunes

```graphql
# Union = "puede ser uno de estos tipos"
# No comparte campos (a diferencia de Interface)
# Útil para resultados de búsqueda, respuestas polimórficas
union SearchResult = Product | User | Category

type Query {
  search(query: String!): [SearchResult!]!
}
```

---

## El Lenguaje de Operaciones

### Query

```graphql
# Forma abreviada (solo para queries simples, sin variables ni fragmentos)
{
  products {
    id
    name
  }
}

# Forma completa — la única que deberías usar en producción
query ListProducts {
  products {
    id
    name
    price
  }
}

# Con argumentos
query GetProduct($productId: ID!) {
  product(id: $productId) {
    id
    name
    price
    category {
      name
    }
  }
}
```

Las variables se pasan como JSON separado del documento GraphQL:

```json
{ "productId": "abc-123" }
```

**Nunca interpoles variables en el string de la query** — es equivalent a SQL injection:

```graphql
# MAL — inyección posible
query { product(id: "${userInput}") { ... } }

# BIEN — variable tipada y validada
query GetProduct($id: ID!) { product(id: $id) { ... } }
```

### Mutation

```graphql
mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    id
    name
    price
    createdAt
  }
}
```

Variables:

```json
{
  "input": {
    "name": "Laptop Pro",
    "price": 1500.0,
    "stock": 10,
    "categoryId": "cat-123"
  }
}
```

**Regla de la spec**: las mutations se ejecutan en SERIE. Si enviás dos mutations en una sola operación, la spec garantiza que se ejecutan en orden:

```graphql
mutation SerialOperations {
  first: deleteProduct(id: "old-id")
  second: createProduct(input: {...}) { id }
  # "second" se ejecuta DESPUÉS de que "first" termine
}
```

### Subscription

```graphql
subscription WatchProduct($id: ID!) {
  productUpdated(id: $id) {
    id
    name
    stock
    updatedAt
  }
}
```

El servidor mantiene una conexión persistente (WebSocket o SSE). Cada vez que el evento ocurre, el servidor envía el resultado del único campo raíz al cliente.

---

## Fragments

### Named Fragments — reutilizar selecciones

```graphql
# Definir el fragment
fragment ProductCardFields on Product {
  id
  name
  price
  category {
    name
  }
}

# Usar con spread operator ...
query {
  products {
    ...ProductCardFields
  }
}

query GetProduct($id: ID!) {
  product(id: $id) {
    ...ProductCardFields
    description # campos adicionales además del fragment
    stock
  }
}
```

### Inline Fragments — para tipos polimórficos

```graphql
# Esenciales con Interfaces y Unions
query Search($q: String!) {
  search(query: $q) {
    __typename # el tipo real del objeto
    ... on Product {
      id
      name
      price
    }

    ... on User {
      id
      email
    }

    ... on Category {
      id
      name
    }
  }
}
```

---

## Directives — modificar comportamiento

Las directives son anotaciones que modifican cómo se ejecuta parte de una query.

```graphql
# @include(if: Boolean!) — incluir el campo solo si la condición es true
query GetProduct($id: ID!, $withCategory: Boolean!) {
  product(id: $id) {
    id
    name
    category @include(if: $withCategory) {
      name
    }
  }
}

# @skip(if: Boolean!) — opuesto de @include
query GetProduct($id: ID!, $skipDescription: Boolean!) {
  product(id: $id) {
    id
    name
    description @skip(if: $skipDescription)
  }
}

# @deprecated — marcar campos como obsoletos en el schema
type Product {
  id: ID!
  name: String!
  price: Float!
  cost: Float @deprecated(reason: "Usar 'price' en su lugar")
}

# @specifiedBy — para custom scalars, apunta a la spec del scalar
scalar DateTime
  @specifiedBy(url: "https://scalars.graphql.org/andimarek/date-time")
```

---

## El Modelo de Ejecución

Cuando llega un request, el servidor GraphQL sigue estos pasos:

```
1. PARSE
   El documento (string GraphQL) es parseado a un AST (Abstract Syntax Tree).
   Errores de sintaxis → respuesta de error inmediata, sin ejecutar nada.

2. VALIDATE
   El AST se valida contra el schema:
   - ¿Existen los tipos y campos pedidos?
   - ¿Los argumentos tienen los tipos correctos?
   - ¿Los fragments son válidos y no tienen ciclos?
   - ¿Las variables están bien tipadas?
   Errores de validación → respuesta de error inmediata.

3. EXECUTE
   Por cada campo del tipo raíz, el servidor invoca el RESOLVER correspondiente.
   Los resolvers reciben 4 argumentos (en todos los lenguajes):

   resolve(obj, args, context, info)
   │        │    │       │       └── metadatos: el campo, el schema, el AST fragment
   │        │    │       └────────── contexto del request: user, DB, dataloaders
   │        │    └────────────────── argumentos del campo: {id: "abc", filter: {...}}
   │        └─────────────────────── objeto padre (resultado del resolver anterior)

4. SERIALIZE
   Los resultados se serializan a JSON y se retornan.
```

### Default resolver

Si no definís un resolver para un campo, el servidor usa el default: retorna `obj[fieldName]`. Por eso en gqlgen podés no definir resolvers para campos simples como `product.name`.

### Non-null bubbling — error propagation

```graphql
type Product {
  id: ID! # non-null
  name: String! # non-null
}

type Query {
  product(id: ID!): Product # nullable (sin !)
}
```

Si el resolver de `name` retorna `null` (o tira un error), y `name` es `String!` (non-null):

- `name` no puede ser null → vuelve al padre `product`
- Si `product` fuera `Product!` (non-null), seguiría subiendo
- Si `product` es nullable → se convierte en null

Resultado:

```json
{
  "data": { "product": null },
  "errors": [{ "message": "...", "path": ["product", "name"] }]
}
```

---

## El Formato de Respuesta

La spec define exactamente cómo debe ser la respuesta:

```json
{
  "data": {
    "product": {
      "id": "abc-123",
      "name": "Laptop Pro",
      "price": 1500.0
    }
  },
  "errors": [
    {
      "message": "No se puede acceder al campo 'secret'",
      "locations": [{ "line": 3, "column": 5 }],
      "path": ["product", "secret"],
      "extensions": {
        "code": "FORBIDDEN",
        "correlationId": "req-xyz-789"
      }
    }
  ]
}
```

Reglas clave:

- `data` y `errors` pueden coexistir → respuesta **parcialmente exitosa**
- `data` es `null` solo si hubo un error en la ejecución del campo raíz
- `errors` es un array (puede haber múltiples errores)
- `extensions` es libre — el servidor puede agregar metadatos (código, correlationId, etc.)

---

## Introspection — el schema se autodocumenta

```graphql
# Consultar el schema completo
query {
  __schema {
    queryType {
      name
    }
    mutationType {
      name
    }
    subscriptionType {
      name
    }
    types {
      name
      kind # SCALAR, OBJECT, INTERFACE, UNION, ENUM, INPUT_OBJECT, LIST, NON_NULL
    }
  }
}

# Consultar un tipo específico
query {
  __type(name: "Product") {
    name
    fields {
      name
      type {
        name
        kind
        ofType {
          name
          kind
        } # para NON_NULL y LIST
      }
      isDeprecated
      deprecationReason
    }
  }
}

# __typename — disponible en cualquier campo, retorna el tipo del objeto
query {
  search(query: "laptop") {
    __typename # retorna "Product" o "User" etc.
    ... on Product {
      name
    }
  }
}
```

**Seguridad**: en producción se recomienda deshabilitar o limitar la introspection para no exponer el schema a atacantes.

```go
// gqlgen — deshabilitar introspection en producción
srv := handler.NewDefaultServer(...)
if os.Getenv("ENV") == "production" {
    srv.AddTransport(transport.POST{})
    // No añadir IntrospectionQuery handler
}
```

---

## N+1 Problem — el mayor problema de rendimiento

```
Query: { products { category { name } } }

Sin DataLoader:
  SELECT * FROM products         → 10 productos
  SELECT * FROM categories WHERE id = 1
  SELECT * FROM categories WHERE id = 2
  SELECT * FROM categories WHERE id = 3
  ... (10 queries) → N+1 = 11 queries totales

Con DataLoader:
  SELECT * FROM products                              → 1 query
  SELECT * FROM categories WHERE id IN (1,2,3,...)   → 1 query (batch)
  → 2 queries totales (sin importar cuántos productos)
```

DataLoader = batch (agrupar IDs) + cache (por request, no global).

---

## Persisted Queries / APQ

En lugar de enviar el string completo de la query en cada request, se envía un hash. Reduce el tamaño del payload y permite cachear en CDN.

```
1. Cliente envía: { "extensions": {"persistedQuery": {"sha256Hash": "abc123"}} }
2. Servidor: no conoce ese hash → responde PersistedQueryNotFound
3. Cliente reenvía: query completa + hash
4. Servidor guarda el hash en caché → 200 OK
5. Próxima vez: solo el hash → hit de caché → OK directo
```

---

## Diferencias Clave vs REST

| Aspecto              | REST                           | GraphQL                             |
| -------------------- | ------------------------------ | ----------------------------------- |
| Endpoints            | Múltiples (`/users`, `/posts`) | Uno solo (`/graphql`)               |
| Datos retornados     | El servidor decide             | El cliente decide                   |
| Overfetching         | Frecuente                      | Eliminado                           |
| Underfetching        | Frecuente (múltiples requests) | Eliminado                           |
| Versionado           | `/v1`, `/v2`...                | Evolución sin versiones             |
| Cache HTTP           | Nativo (GET + CDN)             | Requiere config especial (APQ)      |
| Documentación        | Manual (Swagger, etc.)         | Introspection automática            |
| Error handling       | HTTP status codes              | Siempre 200, errores en `errors[]`  |
| Curva de aprendizaje | Baja                           | Media                               |
| Tooling              | Muy maduro                     | Muy bueno (GraphiQL, Apollo Studio) |
