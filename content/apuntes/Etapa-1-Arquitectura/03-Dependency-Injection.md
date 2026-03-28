# Dependency Injection en Go

DI es el patrón de pasar las dependencias desde afuera en lugar de crearlas adentro.

---

## El problema sin DI

```go
// Sin DI — el usecase construye sus propias dependencias
type productUseCase struct{}

func NewProductUseCase() *productUseCase {
    db, _ := sql.Open("postgres", "postgres://localhost/db")   // ← acoplado a PostgreSQL
    repo := &postgresRepository{db: db}                         // ← acoplado a posgtres repo
    return &productUseCase{repo: repo}
}

// Problemas:
// 1. Imposible testear sin una DB de PostgreSQL real
// 2. Imposible cambiar a otra DB sin tocar el usecase
// 3. La cadena de dependencias se construye dentro del usecase
```

---

## DI manual — el estilo de Go

Go no tiene un framework de DI como Spring (Java). La DI se hace a mano en `main.go`.

```go
// cmd/main.go — el "wiring" (cableado) de todas las dependencias
func main() {
    // 1. Infraestructura (la base)
    db := mustConnectDB(os.Getenv("DATABASE_URL"))
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    emailClient := mail.NewClient(os.Getenv("SMTP_HOST"), os.Getenv("SMTP_USER"), os.Getenv("SMTP_PASS"))

    // 2. Repositories (dependen de infraestructura)
    productRepo := repository.NewPostgresProductRepository(db)
    userRepo := repository.NewPostgresUserRepository(db)

    // 3. Services/NotificationService (dependen de infraestructura)
    notifier := notification.NewEmailNotifier(emailClient)

    // 4. Use Cases (dependen de interfaces de dominio)
    productUC := usecase.NewProductUseCase(productRepo, logger)
    userUC := usecase.NewUserUseCase(userRepo, notifier, logger)

    // 5. Delivery (depende de use cases)
    productHandler := http.NewProductHandler(productUC)
    userHandler := http.NewUserHandler(userUC)

    // 6. Router
    router := setupRouter(productHandler, userHandler)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

---

## Constructor con interfaz — firma correcta

```go
// El constructor acepta la INTERFAZ y retorna la INTERFAZ
func NewProductUseCase(
    repo domain.ProductRepository,   // interfaz
    logger *slog.Logger,             // log es concreto aquí, está bien — es infrastructure
) domain.ProductUseCase {            // retorna interfaz
    return &productUseCase{
        repo:   repo,
        logger: logger,
    }
}

// No esto:
// func NewProductUseCase(repo *postgres.Repository) *productUseCase — demasiado concreto
```

---

## DI en tests — el poder real de DI

```go
// tests/usecase/product_usecase_test.go

// Mock manual del repositorio
type mockProductRepository struct {
    products map[string]*domain.Product
    createErr error
    getByIDErr error
}

func (m *mockProductRepository) GetByID(id string) (*domain.Product, error) {
    if m.getByIDErr != nil {
        return nil, m.getByIDErr
    }
    p, ok := m.products[id]
    if !ok {
        return nil, domain.NewNotFoundError("producto no encontrado")
    }
    return p, nil
}

func (m *mockProductRepository) Create(p *domain.Product) error {
    if m.createErr != nil {
        return m.createErr
    }
    m.products[p.ID] = p
    return nil
}

// ... otros métodos de la interfaz

// Test unitario — sin base de datos, sin red
func TestCreateProduct_ValidInput(t *testing.T) {
    repo := &mockProductRepository{
        products: make(map[string]*domain.Product),
    }
    uc := usecase.NewProductUseCase(repo)

    input := domain.CreateProductInput{
        Name:  "Teclado",
        Price: 89.99,
        Stock: 10,
    }

    product, err := uc.Create(input)

    if err != nil {
        t.Fatalf("error inesperado: %v", err)
    }
    if product.Name != input.Name {
        t.Errorf("esperado %q, got %q", input.Name, product.Name)
    }
    if product.ID == "" {
        t.Error("el product ID no debería estar vacío")
    }
}

func TestCreateProduct_EmptyName(t *testing.T) {
    repo := &mockProductRepository{products: make(map[string]*domain.Product)}
    uc := usecase.NewProductUseCase(repo)

    _, err := uc.Create(domain.CreateProductInput{
        Name:  "",
        Price: 10.0,
    })

    if err == nil {
        t.Fatal("se esperaba error con nombre vacío")
    }

    var appErr *domain.AppError
    if !errors.As(err, &appErr) {
        t.Fatalf("se esperaba *domain.AppError, got %T", err)
    }
    if appErr.Code != domain.ErrCodeInvalidInput {
        t.Errorf("se esperaba código %q, got %q", domain.ErrCodeInvalidInput, appErr.Code)
    }
}
```

---

## Estrategias para mocks

### 1. Mock manual (recomendado para proyectos chicos)

```go
type mockRepo struct {
    products map[string]*domain.Product
    fail     bool
}
func (m *mockRepo) GetByID(id string) (*domain.Product, error) {
    if m.fail {
        return nil, errors.New("error de base de datos simulado")
    }
    // ...
}
```

### 2. mockgen (automático, recomendado en proyectos medianos/grandes)

```bash
go install go.uber.org/mock/mockgen@latest

# Generar mocks desde la interfaz
mockgen -source=internal/domain/product.go \
        -destination=internal/mocks/product_mock.go \
        -package=mocks
```

```go
// Uso en test con gomock
func TestCreate_DBError(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := mocks.NewMockProductRepository(ctrl)
    mockRepo.EXPECT().
        Create(gomock.Any()).
        Return(errors.New("connection refused"))

    uc := usecase.NewProductUseCase(mockRepo)
    _, err := uc.Create(domain.CreateProductInput{Name: "Test", Price: 10})

    if err == nil {
        t.Fatal("se esperaba error")
    }
}
```

---

## Wire (Google) — generador de DI para proyectos grandes

Para proyectos con 50+ componentes, el wiring manual es tedioso. Google Wire genera el código de DI automáticamente.

```go
// wire/wire.go — define cómo cablear dependencias
//go:build wireinject

package wire

import "github.com/google/wire"

func InitializeAPI() (*http.Server, error) {
    wire.Build(
        db.NewConnection,
        repository.NewPostgresProductRepository,
        usecase.NewProductUseCase,
        handler.NewProductHandler,
        setupServer,
    )
    return &http.Server{}, nil
}
```

```bash
go run github.com/google/wire/cmd/wire ./...
# Genera wire_gen.go con todo el código de DI
```

> Para proyectos chicos/medianos, DI manual es suficiente y más legible. Wire empieza a valer cuando tenés >20 componentes en el wiring.

---

## Práctica: Novato vs Profesional

### Novato — crear dependencias adentro

```go
type OrderUseCase struct{}

func (u *OrderUseCase) PlaceOrder(userID, productID string, qty int) error {
    // crea sus propias dependencias — imposible de testear
    db, _ := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    repo := &postgresOrderRepo{db: db}
    emailClient := &SMTPClient{host: "smtp.gmail.com"}

    order := createOrder(userID, productID, qty)
    repo.Save(order)
    emailClient.Send(userID, "Orden confirmada")
    return nil
}
```

### Profesional — dependencias inyectadas

```go
// Interfaces en el dominio
type OrderRepository interface {
    Save(o *Order) error
    GetByID(id string) (*Order, error)
}

type Notifier interface {
    Notify(userID string, message string) error
}

// UseCase recibe interfaces
type orderUseCase struct {
    repo     OrderRepository
    notifier Notifier
    logger   *slog.Logger
}

func NewOrderUseCase(repo OrderRepository, notifier Notifier, logger *slog.Logger) OrderUseCase {
    return &orderUseCase{repo: repo, notifier: notifier, logger: logger}
}

func (u *orderUseCase) PlaceOrder(ctx context.Context, input PlaceOrderInput) (*Order, error) {
    // validación de negocio
    order := &Order{
        ID:        uuid.New().String(),
        UserID:    input.UserID,
        ProductID: input.ProductID,
        Quantity:  input.Quantity,
    }

    if err := u.repo.Save(order); err != nil {
        return nil, fmt.Errorf("guardando orden: %w", err)
    }

    if err := u.notifier.Notify(input.UserID, "Orden confirmada"); err != nil {
        // loguear pero no fallar — la notificación es secundaria
        u.logger.Warn("fallo al notificar", "error", err, "userID", input.UserID)
    }

    return order, nil
}
```
