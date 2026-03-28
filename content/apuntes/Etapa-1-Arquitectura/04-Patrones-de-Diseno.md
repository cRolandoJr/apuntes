# Patrones de Diseño en Go

Los patrones de diseño son soluciones reutilizables a problemas comunes. En Go se expresan de forma más simple que en Java/C# porque Go tiene interfaces implícitas y funciones como valores.

---

## Repository Pattern

Ya cubierto en Clean Architecture. El repositorio abstrae el acceso a datos.

```go
// 1. Interfaz en el dominio
type ProductRepository interface {
    GetByID(id string) (*Product, error)
    GetAll() ([]*Product, error)
    Create(p *Product) error
    Update(p *Product) error
    Delete(id string) error
}

// 2. Implementaciones múltiples
type InMemoryRepo struct{ data map[string]*Product }
type PostgresRepo struct{ db *pgxpool.Pool }
type CachedRepo struct {
    cache    *redis.Client
    fallback ProductRepository  // decorador — envuelve otra implementación
}

// 3. El usecase no sabe cuál se usa
type productUseCase struct {
    repo ProductRepository
}
```

---

## Factory Pattern

Crear objetos sin exponer la lógica de construcción.

```go
// Factory simple — función constructora
func NewProduct(name string, price float64, stock int) (*Product, error) {
    if strings.TrimSpace(name) == "" {
        return nil, NewInvalidInputError("nombre requerido")
    }
    if price <= 0 {
        return nil, NewInvalidInputError("precio debe ser mayor a cero")
    }
    return &Product{
        ID:        uuid.New().String(),
        Name:      name,
        Price:     price,
        Stock:     stock,
        CreatedAt: time.Now(),
    }, nil
}

// Abstract Factory — crear familias de objetos relacionados
type RepositoryFactory interface {
    NewProductRepository() ProductRepository
    NewUserRepository() UserRepository
    NewOrderRepository() OrderRepository
}

type PostgresRepositoryFactory struct{ db *pgxpool.Pool }

func (f *PostgresRepositoryFactory) NewProductRepository() ProductRepository {
    return &postgresProductRepo{db: f.db}
}
func (f *PostgresRepositoryFactory) NewUserRepository() UserRepository {
    return &postgresUserRepo{db: f.db}
}

type InMemoryRepositoryFactory struct{}

func (f *InMemoryRepositoryFactory) NewProductRepository() ProductRepository {
    return &inMemoryProductRepo{data: make(map[string]*Product)}
}
```

---

## Functional Options Pattern

Para configurar structs con muchas opciones sin constructores frágiles. Ver también [[02-Funciones-y-Metodos]].

```go
type Server struct {
    host         string
    port         int
    timeout      time.Duration
    maxConns     int
    readTimeout  time.Duration
    writeTimeout time.Duration
    tls          bool
    logger       *slog.Logger
}

type ServerOption func(*Server)

func WithHost(host string) ServerOption {
    return func(s *Server) { s.host = host }
}
func WithPort(port int) ServerOption {
    return func(s *Server) { s.port = port }
}
func WithTLS() ServerOption {
    return func(s *Server) { s.tls = true }
}
func WithLogger(logger *slog.Logger) ServerOption {
    return func(s *Server) { s.logger = logger }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{  // defaults sensatos
        host:         "0.0.0.0",
        port:         8080,
        timeout:      30 * time.Second,
        maxConns:     1000,
        readTimeout:  10 * time.Second,
        writeTimeout: 10 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Uso legible
srv := NewServer(
    WithPort(9090),
    WithTLS(),
    WithLogger(logger),
)
```

---

## Strategy Pattern

Definir una familia de algoritmos, encapsularlos e intercambiarlos.

```go
// Notificaciones con diferentes canales
type NotificationStrategy interface {
    Send(userID, message string) error
}

type EmailStrategy struct{ client *mail.Client }
func (s *EmailStrategy) Send(userID, message string) error {
    email := lookupEmail(userID)
    return s.client.Send(email, message)
}

type SMSStrategy struct{ client *sms.Client }
func (s *SMSStrategy) Send(userID, message string) error {
    phone := lookupPhone(userID)
    return s.client.Send(phone, message)
}

type PushStrategy struct{ client *push.Client }
func (s *PushStrategy) Send(userID, message string) error {
    token := lookupDeviceToken(userID)
    return s.client.Send(token, message)
}

// Multi-channel: combinar estrategias
type MultiStrategy struct{ strategies []NotificationStrategy }
func (m *MultiStrategy) Send(userID, message string) error {
    var errs []error
    for _, s := range m.strategies {
        if err := s.Send(userID, message); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)  // Go 1.20+
}

// Uso con DI
notifier := &MultiStrategy{strategies: []NotificationStrategy{
    &EmailStrategy{client: emailClient},
    &PushStrategy{client: pushClient},
}}
uc := NewOrderUseCase(repo, notifier)
```

---

## Observer Pattern (Event-Driven)

Notificar a múltiples interesados cuando algo sucede.

```go
// Event types
type EventType string
const (
    EventProductCreated EventType = "product.created"
    EventProductDeleted EventType = "product.deleted"
    EventOrderPlaced    EventType = "order.placed"
)

type Event struct {
    Type    EventType
    Payload any
    OccurredAt time.Time
}

// El bus de eventos
type EventHandler func(ctx context.Context, event Event) error

type EventBus struct {
    mu       sync.RWMutex
    handlers map[EventType][]EventHandler
}

func NewEventBus() *EventBus {
    return &EventBus{handlers: make(map[EventType][]EventHandler)}
}

func (b *EventBus) Subscribe(eventType EventType, handler EventHandler) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.handlers[eventType] = append(b.handlers[eventType], handler)
}

func (b *EventBus) Publish(ctx context.Context, event Event) error {
    b.mu.RLock()
    handlers := b.handlers[event.Type]
    b.mu.RUnlock()

    for _, h := range handlers {
        if err := h(ctx, event); err != nil {
            return fmt.Errorf("manejando evento %s: %w", event.Type, err)
        }
    }
    return nil
}

// Uso en el usecase — con DI del bus
type productUseCase struct {
    repo     ProductRepository
    eventBus *EventBus
}

func (u *productUseCase) Create(ctx context.Context, input CreateProductInput) (*Product, error) {
    // ... crear el producto ...

    u.eventBus.Publish(ctx, Event{
        Type:       EventProductCreated,
        Payload:    product,
        OccurredAt: time.Now(),
    })

    return product, nil
}

// Handlers independientes suscritos al bus
func RegisterHandlers(bus *EventBus, emailClient *mail.Client, analyticsClient *analytics.Client) {
    bus.Subscribe(EventProductCreated, func(ctx context.Context, e Event) error {
        p := e.Payload.(*Product)
        return emailClient.Send("admin@empresa.com", "Nuevo producto: "+p.Name)
    })

    bus.Subscribe(EventProductCreated, func(ctx context.Context, e Event) error {
        p := e.Payload.(*Product)
        return analyticsClient.Track("product_created", map[string]any{"id": p.ID})
    })
}
```

---

## Decorator Pattern

Envolver un objeto para agregar comportamiento sin modificar el original.

```go
// Decorator 1: cache para el repository
type cachedProductRepository struct {
    cache    map[string]*Product
    mu       sync.RWMutex
    fallback ProductRepository  // el repo real
}

func NewCachedRepository(repo ProductRepository) ProductRepository {
    return &cachedProductRepository{
        cache:    make(map[string]*Product),
        fallback: repo,
    }
}

func (r *cachedProductRepository) GetByID(id string) (*Product, error) {
    r.mu.RLock()
    if p, ok := r.cache[id]; ok {
        r.mu.RUnlock()
        return p, nil  // cache hit
    }
    r.mu.RUnlock()

    // cache miss — ir al repo real
    p, err := r.fallback.GetByID(id)
    if err != nil {
        return nil, err
    }

    r.mu.Lock()
    r.cache[id] = p
    r.mu.Unlock()

    return p, nil
}

func (r *cachedProductRepository) Create(p *Product) error {
    err := r.fallback.Create(p)
    if err == nil {
        r.mu.Lock()
        r.cache[p.ID] = p
        r.mu.Unlock()
    }
    return err
}

// Decorator 2: logging para el usecase
type loggedProductUseCase struct {
    wrapped ProductUseCase
    logger  *slog.Logger
}

func (l *loggedProductUseCase) Create(input CreateProductInput) (*Product, error) {
    start := time.Now()
    product, err := l.wrapped.Create(input)
    l.logger.Info("Create",
        "duration", time.Since(start),
        "error", err,
        "productID", func() string {
            if product != nil {
                return product.ID
            }
            return ""
        }(),
    )
    return product, err
}

// En main.go — apilar decoradores
repo := repository.NewPostgresProductRepository(db)
cachedRepo := repository.NewCachedRepository(repo)  // agregar cache

uc := usecase.NewProductUseCase(cachedRepo)
loggedUC := usecase.NewLoggedUseCase(uc, logger)   // agregar logging

handler := http.NewProductHandler(loggedUC)
```

---

## Middleware Pattern (HTTP)

Cadena de handlers que se aplican en orden.

```go
// Firma estándar de middleware en Go
type Middleware func(http.Handler) http.Handler

// Middleware de logging
func LoggingMiddleware(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

            next.ServeHTTP(wrapped, r)

            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "status", wrapped.statusCode,
                "duration", time.Since(start),
            )
        })
    }
}

// Middleware de autenticación
func AuthMiddleware(tokenService TokenService) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("Authorization")
            claims, err := tokenService.Validate(token)
            if err != nil {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
            ctx := context.WithValue(r.Context(), claimsKey{}, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Aplicar middleware (cadena)
func Chain(middlewares ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}

// Uso en main.go
handler := Chain(
    LoggingMiddleware(logger),
    AuthMiddleware(tokenSvc),
    RateLimitMiddleware(100),
)(router)
```

---

## Singleton con sync.Once

```go
// Thread-safe, lazy initialization
type dbPool struct{ pool *pgxpool.Pool }

var (
    instance *dbPool
    once     sync.Once
)

func GetDBPool(dsn string) *pgxpool.Pool {
    once.Do(func() {
        pool, err := pgxpool.New(context.Background(), dsn)
        if err != nil {
            log.Fatalf("no se pudo conectar a la DB: %v", err)
        }
        instance = &dbPool{pool: pool}
    })
    return instance.pool
}
```

> En Go, generalmente se prefiere pasar las dependencias explícitamente (DI) en lugar de usar Singleton. Singleton es aceptable solo para conexiones de infraestructura creadas una vez en main.
