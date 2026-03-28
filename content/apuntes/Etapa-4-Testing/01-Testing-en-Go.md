# Testing en Go

El testing en Go está integrado en el lenguaje. `go test` es el comando, `testing` es el paquete. No necesitás frameworks externos para empezar.

---

## Convenciones fundamentales

```
internal/
└── usecase/
    ├── product_usecase.go
    └── product_usecase_test.go  ← mismo paquete o _test
```

```go
// Reglas de nombramiento
func TestNombreDelComponente_Escenario(t *testing.T) {}

// Ejemplos:
func TestProductUseCase_Create_Succeeds(t *testing.T) {}
func TestProductUseCase_Create_FailsWhenNameEmpty(t *testing.T) {}
func TestGetByID_NotFound_ReturnsError(t *testing.T) {}
```

---

## Table-driven tests — el patrón Go estándar

```go
func TestProductUseCase_Create(t *testing.T) {
    tests := []struct {
        name        string
        input       domain.CreateProductInput
        setupMock   func(*mockProductRepository)
        wantErr     bool
        wantErrCode domain.ErrorCode
        wantProduct *domain.Product
    }{
        {
            name: "crea producto correctamente",
            input: domain.CreateProductInput{
                Name:       "Laptop Pro",
                Price:      1500.00,
                Stock:      10,
                CategoryID: "cat-id-1",
            },
            setupMock: func(m *mockProductRepository) {
                m.On("Create", mock.Anything, mock.AnythingOfType("*domain.Product")).
                    Return(&domain.Product{ID: "new-id", Name: "Laptop Pro"}, nil)
            },
            wantErr:     false,
            wantProduct: &domain.Product{ID: "new-id", Name: "Laptop Pro"},
        },
        {
            name: "falla con nombre vacío",
            input: domain.CreateProductInput{
                Name:  "", // inválido
                Price: 100.00,
            },
            setupMock:   func(m *mockProductRepository) {}, // no se llama al repo
            wantErr:     true,
            wantErrCode: domain.ErrCodeInvalidInput,
        },
        {
            name: "falla con precio negativo",
            input: domain.CreateProductInput{
                Name:  "Producto",
                Price: -50.00, // inválido
            },
            setupMock:   func(m *mockProductRepository) {},
            wantErr:     true,
            wantErrCode: domain.ErrCodeInvalidInput,
        },
        {
            name: "falla cuando el repo retorna error",
            input: domain.CreateProductInput{
                Name:  "Producto",
                Price: 100.00,
            },
            setupMock: func(m *mockProductRepository) {
                m.On("Create", mock.Anything, mock.Anything).
                    Return(nil, errors.New("db error"))
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockRepo := &mockProductRepository{}
            tt.setupMock(mockRepo)

            uc := usecase.NewProductUseCase(mockRepo)

            // Act
            got, err := uc.Create(context.Background(), tt.input)

            // Assert
            if tt.wantErr {
                assert.Error(t, err)
                if tt.wantErrCode != "" {
                    var appErr *domain.AppError
                    require.ErrorAs(t, err, &appErr)
                    assert.Equal(t, tt.wantErrCode, appErr.Code)
                }
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.wantProduct.ID,   got.ID)
            assert.Equal(t, tt.wantProduct.Name, got.Name)

            mockRepo.AssertExpectations(t)
        })
    }
}
```

---

## assert vs require (testify)

```bash
go get github.com/stretchr/testify
```

| Función                           | Comportamiento al fallar                               |
| --------------------------------- | ------------------------------------------------------ |
| `assert.Equal(t, expected, got)`  | Marca como fallido, **continúa** el test               |
| `require.Equal(t, expected, got)` | Marca como fallido, **detiene** el test inmediatamente |
| `assert.Error(t, err)`            | Continúa                                               |
| `require.NoError(t, err)`         | Detiene — útil antes de usar el resultado              |

Regla: usar `require` cuando el siguiente código depende del resultado. Usar `assert` para verificaciones independientes.

---

## Mocks — dos enfoques

### Mock manual — para interfaces simples

```go
// En el mismo paquete de test o en un subpaquete testutil/

type mockProductRepository struct {
    products map[string]*domain.Product
    err      error // para simular errores
}

func (m *mockProductRepository) GetByID(_ context.Context, id string) (*domain.Product, error) {
    if m.err != nil {
        return nil, m.err
    }
    p, ok := m.products[id]
    if !ok {
        return nil, domain.NewNotFoundError("no encontrado")
    }
    return p, nil
}

func (m *mockProductRepository) Create(_ context.Context, p *domain.Product) (*domain.Product, error) {
    if m.err != nil {
        return nil, m.err
    }
    p.ID = "generated-id"
    m.products[p.ID] = p
    return p, nil
}

// Uso
func TestGetByID(t *testing.T) {
    mock := &mockProductRepository{
        products: map[string]*domain.Product{
            "id-1": {ID: "id-1", Name: "Test"},
        },
    }
    uc := usecase.NewProductUseCase(mock)
    // ...
}
```

### Mock con testify/mock — para interfaces complejas

```go
// Se puede generar con mockgen, pero testify/mock es más idiomático en proyectos pequeños

type MockProductRepository struct {
    mock.Mock
}

func (m *MockProductRepository) GetByID(ctx context.Context, id string) (*domain.Product, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.Product), args.Error(1)
}

// En el test
mockRepo := new(MockProductRepository)
mockRepo.On("GetByID", mock.Anything, "p-1").
    Return(&domain.Product{ID: "p-1"}, nil)
mockRepo.On("GetByID", mock.Anything, "no-existe").
    Return(nil, domain.NewNotFoundError("no encontrado"))

// Al final del test
mockRepo.AssertExpectations(t) // verifica que todos los On() fueron llamados
```

### mockgen — generar mocks automáticamente

```bash
go install go.uber.org/mock/mockgen@latest

# Generar mock de una interfaz
mockgen -source=internal/domain/product.go -destination=internal/testutil/mock_product_repository.go -package=testutil
```

---

## Tests de repositorio con testcontainers

Tests de integración reales contra una base de datos PostgreSQL en Docker.

```bash
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

```go
// internal/repository/postgres_product_repository_test.go

package repository_test

import (
    "context"
    "testing"

    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/stretchr/testify/require"
    "gestion_productos/internal/domain"
    "gestion_productos/internal/repository"
    pgconn "gestion_productos/internal/infra/postgres"
)

func setupTestDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := context.Background()

    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections"),
        ),
    )
    require.NoError(t, err)

    t.Cleanup(func() {
        _ = container.Terminate(context.Background())
    })

    dsn, err := container.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    pool, err := pgconn.NewPool(ctx, pgconn.Config{DSN: dsn, MaxConnections: 5})
    require.NoError(t, err)

    // Ejecutar migraciones
    err = migrations.Run(dsn)
    require.NoError(t, err)

    return pool
}

func TestPostgresProductRepository_Create(t *testing.T) {
    pool := setupTestDB(t)
    repo := repository.NewPostgresProductRepository(pool)

    input := &domain.Product{
        Name:       "Test Product",
        Price:      100.00,
        Stock:      10,
        CategoryID: "cat-id",
    }

    got, err := repo.Create(context.Background(), input)

    require.NoError(t, err)
    assert.NotEmpty(t, got.ID)
    assert.Equal(t, "Test Product", got.Name)
    assert.NotZero(t, got.CreatedAt)
}
```

---

## Tests de handlers HTTP

```go
func TestProductHandler_Create(t *testing.T) {
    mockUC := &MockProductUseCase{}
    handler := httphandler.NewProductHandler(mockUC)

    body := strings.NewReader(`{"name":"Laptop","price":1500,"stock":5}`)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/products", body)
    req.Header.Set("Content-Type", "application/json")

    // Inyectar claims al contexto (como lo haría el middleware)
    ctx := middleware.WithClaims(req.Context(), &domain.Claims{UserID: "u-1", Role: domain.RoleAdmin})
    req = req.WithContext(ctx)

    w := httptest.NewRecorder()

    mockUC.On("Create", mock.Anything, mock.AnythingOfType("domain.CreateProductInput")).
        Return(&domain.Product{ID: "new-id", Name: "Laptop"}, nil)

    handler.Create(w, req)

    resp := w.Result()
    assert.Equal(t, http.StatusCreated, resp.StatusCode)

    var result map[string]interface{}
    _ = json.NewDecoder(resp.Body).Decode(&result)
    assert.Equal(t, "new-id", result["id"])
}
```

---

## Comandos esenciales

```bash
# Correr todos los tests
go test ./...

# Con output detallado
go test -v ./...

# Un paquete específico
go test -v ./internal/usecase/...

# Tests de integración (con tag)
go test -v -tags integration ./...

# Con detector de race conditions
go test -race ./...

# Con cobertura
go test -race -coverprofile=coverage.out ./...
go tool cover -html=coverage.out  # ver en browser

# Correr un test específico
go test -run TestProductUseCase_Create_Succeeds ./internal/usecase/

# Correr subtests
go test -run "TestProductUseCase/crea producto correctamente" ./...

# Benchmark
go test -bench=. -benchmem ./...

# Timeout por paquete
go test -timeout 30s ./...
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Tests sin table-driven — duplicación masiva
func TestCreate1(t *testing.T) { /* caso 1 */ }
func TestCreate2(t *testing.T) { /* caso 2 */ }
func TestCreate3(t *testing.T) { /* caso 3 */ }

// Solo testean el happy path
// Sin mocks — tests dependen de la DB real
// assert sin require — panic en el siguiente uso de nil
```

### Profesional

```go
// Table-driven: N casos en 1 función
// Mocks para la capa que no es la que se testea
// require.NoError antes de usar el resultado
// t.Helper() en funciones de setup
// Tests de integración separados con tags
// -race en CI siempre
```
