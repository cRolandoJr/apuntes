# GraphQL con gqlgen

gqlgen es el generador de código GraphQL más maduro para Go. Enfoque schema-first: primero escribís el schema `.graphqls`, luego generás el código.

---

## Setup inicial

```bash
go get github.com/99designs/gqlgen
go run github.com/99designs/gqlgen init  # scaffolding
```

### Estructura generada

```
graph/
├── schema.graphqls         ← TÚ escribís esto
├── generated/
│   └── generated.go        ← generado (no tocar)
└── model/
    └── models_gen.go       ← modelos generados (o usar autobind)
internal/delivery/graphql/
└── resolver.go             ← TÚ implementás esto
gqlgen.yml                 ← configuración del generador
```

---

## gqlgen.yml — configuración

```yaml
schema:
  - graph/*.graphqls

exec:
  filename: graph/generated/generated.go
  package: generated

model:
  filename: graph/model/models_gen.go
  package: model

resolver:
  layout: single-file
  dir: internal/delivery/graphql
  package: graphql
  filename: resolver.go
  type: Resolver

# autobind — usa tus propios tipos en lugar de generar nuevos
autobind:
  - "gestion_productos/internal/domain"
```

---

## Schema — conceptos fundamentales

```graphql
# Tipos de dato
type Product {
  id: ID! # ! = non-null (obligatorio)
  name: String!
  description: String!
  price: Float!
  stock: Int!
  category: Category # nullable — puede ser null
  tags: [String!]! # lista non-null de strings non-null
  createdAt: String!
}

type Category {
  id: ID!
  name: String!
  products: [Product!]!
}

# Inputs — tipos para argumentos de mutaciones
input CreateProductInput {
  name: String!
  description: String!
  price: Float!
  stock: Int!
  categoryId: ID!
}

input UpdateProductInput {
  name: String # nullable = opcional
  description: String
  price: Float
  stock: Int
}

# Queries — operaciones de lectura
type Query {
  products(name: String, categoryId: ID, page: Int, pageSize: Int): [Product!]!

  product(id: ID!): Product!
}

# Mutations — operaciones de escritura
type Mutation {
  createProduct(input: CreateProductInput!): Product!
  updateProduct(id: ID!, input: UpdateProductInput!): Product!
  deleteProduct(id: ID!): Boolean!
}

# Enums
enum ProductOrderBy {
  NAME
  PRICE
  CREATED_AT
}

# Scalars personalizados (registrar en Go también)
scalar DateTime
scalar Decimal
```

---

## Resolver — implementar las operaciones

```go
// internal/delivery/graphql/resolver.go

package graphql

import (
    "context"
    "gestion_productos/graph/generated"
    "gestion_productos/internal/domain"
    "time"
)

type Resolver struct {
    ProductUC  domain.ProductUseCase
    CategoryUC domain.CategoryUseCase
}

// Los resolvers se separan por tipo GraphQL
func (r *Resolver) Mutation() generated.MutationResolver { return &mutationResolver{r} }
func (r *Resolver) Query() generated.QueryResolver       { return &queryResolver{r} }
func (r *Resolver) Product() generated.ProductResolver   { return &productResolver{r} }

type mutationResolver struct{ *Resolver }
type queryResolver struct{ *Resolver }
type productResolver struct{ *Resolver }

// Query resolvers
func (r *queryResolver) Products(ctx context.Context, name *string) ([]*domain.Product, error) {
    filter := ""
    if name != nil {
        filter = *name
    }
    return r.ProductUC.GetAll(filter)
}

func (r *queryResolver) Product(ctx context.Context, id string) (*domain.Product, error) {
    return r.ProductUC.GetByID(id)
}

// Mutation resolvers
func (r *mutationResolver) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    return r.ProductUC.Create(input)
}

func (r *mutationResolver) UpdateProduct(ctx context.Context, id string, input domain.UpdateProductInput) (*domain.Product, error) {
    return r.ProductUC.Update(id, input)
}

func (r *mutationResolver) DeleteProduct(ctx context.Context, id string) (bool, error) {
    err := r.ProductUC.Delete(id)
    return err == nil, err
}

// Field resolver — para campos del tipo Product que necesitan lógica
func (r *productResolver) CreatedAt(ctx context.Context, obj *domain.Product) (string, error) {
    return obj.CreatedAt.Format(time.RFC3339), nil
}

// Field resolver para relaciones (evita N+1 con DataLoader)
func (r *productResolver) Category(ctx context.Context, obj *domain.Product) (*domain.Category, error) {
    return r.CategoryUC.GetByID(obj.CategoryID)
}
```

---

## Error handling en GraphQL

GraphQL no usa HTTP status codes. Los errores van en el campo `errors` de la respuesta.

```go
// cmd/main.go — configurar el error presenter
srv.SetErrorPresenter(func(ctx context.Context, e error) *gqlerror.Error {
    err := gqlgraphql.DefaultErrorPresenter(ctx, e)

    correlationID := uuid.New().String()
    log.Printf("[%s] %v", correlationID, e)

    var appErr *domain.AppError
    if errors.As(e, &appErr) {
        err.Message = appErr.Message
        err.Extensions = map[string]interface{}{
            "code":          string(appErr.Code),
            "correlationId": correlationID,
        }
    } else {
        err.Message = "error interno del servidor"
        err.Extensions = map[string]interface{}{
            "code":          "INTERNAL_ERROR",
            "correlationId": correlationID,
        }
    }

    return err
})
```

Respuesta de error:

```json
{
  "errors": [
    {
      "message": "producto no encontrado",
      "extensions": {
        "code": "NOT_FOUND",
        "correlationId": "abc-123"
      }
    }
  ]
}
```

---

## El problema N+1 — y DataLoader

```graphql
# Esta query puede causar N+1
query {
  products {
    # 1 query para listar productos
    id
    name
    category {
      # N queries, una por producto → N+1
      name
    }
  }
}
```

### Solución con DataLoader

```go
import "github.com/graph-gophers/dataloader/v7"

// Batch function — recibe todos los IDs juntos, hace UNA query
func batchGetCategories(ctx context.Context, keys []string) []*dataloader.Result[*domain.Category] {
    // Una sola query para todos los IDs
    categories, err := categoryRepo.GetByIDs(keys)

    results := make([]*dataloader.Result[*domain.Category], len(keys))
    catMap := make(map[string]*domain.Category)
    for _, c := range categories {
        catMap[c.ID] = c
    }

    for i, key := range keys {
        if cat, ok := catMap[key]; ok {
            results[i] = &dataloader.Result[*domain.Category]{Data: cat}
        } else {
            results[i] = &dataloader.Result[*domain.Category]{Error: domain.NewNotFoundError("categoría no encontrada")}
        }
    }
    return results
}

// Tipo de key para el context
type LoaderKey struct{}

// En el resolver — usar el dataloader del context
func (r *productResolver) Category(ctx context.Context, obj *domain.Product) (*domain.Category, error) {
    loader := ctx.Value(LoaderKey{}).(*dataloader.Loader[string, *domain.Category])
    return loader.Load(ctx, obj.CategoryID)()
}
```

---

## Autenticación en GraphQL

```go
// Middleware HTTP para validar el token antes de llegar al resolver
func graphqlAuthMiddleware(tokenSvc TokenService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")

            if token != "" {
                claims, err := tokenSvc.Validate(token)
                if err == nil {
                    ctx := context.WithValue(r.Context(), claimsKey{}, claims)
                    r = r.WithContext(ctx)
                }
                // Si el token es inválido, simplemente no inyectamos claims
                // El resolver verificará si necesita autenticación
            }

            next.ServeHTTP(w, r)
        })
    }
}

// Helper para obtener claims del contexto en resolvers
func GetClaims(ctx context.Context) (*Claims, bool) {
    claims, ok := ctx.Value(claimsKey{}).(*Claims)
    return claims, ok
}

// En el resolver — verificar autenticación
func (r *mutationResolver) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    claims, ok := GetClaims(ctx)
    if !ok {
        return nil, domain.NewUnauthorizedError("autenticación requerida")
    }
    if !claims.HasPermission(domain.PermWrite) {
        return nil, domain.NewForbiddenError("no tenés permiso para crear productos")
    }
    return r.ProductUC.Create(input)
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Lógica de negocio en el resolver — viola Clean Architecture
func (r *mutationResolver) CreateProduct(ctx context.Context, input model.CreateProductInput) (*model.Product, error) {
    if input.Name == "" {
        return nil, errors.New("nombre requerido")
    }
    // SQL directo en el resolver
    row := r.db.QueryRow("INSERT INTO products ...")
    // ...
}
```

### Profesional

```go
// Resolver solo traduce GraphQL → dominio → GraphQL
// Toda la lógica está en el usecase
func (r *mutationResolver) CreateProduct(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    return r.ProductUC.Create(input)
    // si hay error, el error presenter lo mapea a gqlerror.Error con código tipado
}
```
