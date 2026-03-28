# HTTP Fundamentos en Go

Go tiene un servidor HTTP de alta performance en la stdlib. No necesitás un framework para levantar una API. Los frameworks (chi, gin, echo) agregan routing conveniente, pero la base es `net/http`.

---

## Handler básico

```go
// http.Handler es una interfaz con un solo método
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// http.HandlerFunc es un adaptador que permite usar funciones como handlers
func home(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)              // status code (por defecto es 200)
    w.Header().Set("Content-Type", "text/plain")
    fmt.Fprintln(w, "Hola mundo")
}

func main() {
    http.HandleFunc("/", home)
    http.HandleFunc("/products", productsHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## Server con configuración — NO usar defaults en producción

```go
// El servidor por defecto (http.ListenAndServe) no tiene timeouts
// En producción SIEMPRE configurar timeouts para evitar "slow loris" attacks

srv := &http.Server{
    Addr:         ":8080",
    Handler:      router,
    ReadTimeout:  10 * time.Second,   // tiempo máximo para leer el request completo
    WriteTimeout: 30 * time.Second,   // tiempo máximo para escribir el response
    IdleTimeout:  120 * time.Second,  // tiempo de keep-alive entre requests
    MaxHeaderBytes: 1 << 20,          // 1 MB máximo en headers
}

log.Fatal(srv.ListenAndServe())
```

---

## ServeMux — router de la stdlib

```go
mux := http.NewServeMux()

// Go 1.22+ soporta method y path params en la stdlib
mux.HandleFunc("GET /products", listProducts)
mux.HandleFunc("POST /products", createProduct)
mux.HandleFunc("GET /products/{id}", getProduct)    // path param
mux.HandleFunc("PUT /products/{id}", updateProduct)
mux.HandleFunc("DELETE /products/{id}", deleteProduct)

// Go 1.22: extraer path param
func getProduct(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // nuevo en Go 1.22
    // ...
}
```

### Chi — la alternativa más popular

```go
import "github.com/go-chi/chi/v5"

r := chi.NewRouter()
r.Use(middleware.Logger)        // middleware global
r.Use(middleware.Recoverer)

r.Get("/products", listProducts)
r.Post("/products", createProduct)
r.Route("/products/{id}", func(r chi.Router) {
    r.Get("/", getProduct)
    r.Put("/", updateProduct)
    r.Delete("/", deleteProduct)
})

// En el handler:
id := chi.URLParam(r, "id")
```

---

## ResponseWriter — escribir respuestas

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)  // IMPORTANTE: SetHeader ANTES de WriteHeader
    json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}

// Uso
func createProduct(w http.ResponseWriter, r *http.Request) {
    var input CreateProductInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "input inválido")
        return
    }

    product, err := useCase.Create(input)
    if err != nil {
        status, msg := mapErrorToHTTP(err)
        writeError(w, status, msg)
        return
    }

    writeJSON(w, http.StatusCreated, product)
}
```

---

## Middleware

```go
// Middleware de logging
func logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Middleware para CORS
func cors(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Pasando datos entre middleware con context
type contextKey string

func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := uuid.New().String()
        ctx := context.WithValue(r.Context(), contextKey("requestID"), id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getRequestID(r *http.Request) string {
    id, _ := r.Context().Value(contextKey("requestID")).(string)
    return id
}
```

---

## Handler como struct — inyección de dependencias

```go
// Patrón profesional: handler como struct con sus dependencias
type ProductHandler struct {
    useCase domain.ProductUseCase
    logger  *slog.Logger
}

func NewProductHandler(uc domain.ProductUseCase, logger *slog.Logger) *ProductHandler {
    return &ProductHandler{useCase: uc, logger: logger}
}

// Métodos del handler — acceso a las dependencias del struct
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var input domain.CreateProductInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        h.logger.Warn("body inválido", "error", err)
        writeError(w, http.StatusBadRequest, "formato de request inválido")
        return
    }

    product, err := h.useCase.Create(input)
    if err != nil {
        status, msg := mapDomainError(err)
        h.logger.Error("CreateProduct", "error", err, "input", input)
        writeError(w, status, msg)
        return
    }

    writeJSON(w, http.StatusCreated, product)
}

func (h *ProductHandler) RegisterRoutes(r chi.Router) {
    r.Get("/", h.List)
    r.Post("/", h.Create)
    r.Get("/{id}", h.GetByID)
    r.Put("/{id}", h.Update)
    r.Delete("/{id}", h.Delete)
}

// En main.go
productHandler := NewProductHandler(productUC, logger)
r.Route("/api/v1/products", productHandler.RegisterRoutes)
```

---

## Graceful Shutdown — apagado controlado

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}

    // Lanzar servidor en goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("servidor: %v", err)
        }
    }()

    // Esperar señal de shutdown (Ctrl+C o SIGTERM de Kubernetes)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("iniciando graceful shutdown...")

    // Dar tiempo a los requests en vuelo para terminar
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("shutdown forzado: %v", err)
    }

    log.Println("servidor apagado correctamente")
}
```

---

## Leer datos del request

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Query params: /products?page=2&limit=20
    page := r.URL.Query().Get("page")
    limit := r.URL.Query().Get("limit")

    // Path params (Go 1.22)
    id := r.PathValue("id")

    // Headers
    authHeader := r.Header.Get("Authorization")
    contentType := r.Header.Get("Content-Type")

    // Body JSON
    var input struct {
        Name  string  `json:"name"`
        Price float64 `json:"price"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        // ...
    }
    defer r.Body.Close()

    // Form data
    r.ParseForm()
    name := r.FormValue("name")

    // Multipart (file upload)
    r.ParseMultipartForm(10 << 20)  // 10 MB máximo en memoria
    file, header, err := r.FormFile("upload")
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
func main() {
    http.HandleFunc("/products", func(w http.ResponseWriter, r *http.Request) {
        // Todo mezclado: routing, lógica, DB
        if r.Method == "GET" {
            rows, _ := db.Query("SELECT * FROM products")
            // ...
        } else if r.Method == "POST" {
            // sin validación del Content-Type
            var p Product
            json.NewDecoder(r.Body).Decode(&p)  // error ignorado
            db.Exec("INSERT INTO ...")
        }
    })
    http.ListenAndServe(":8080", nil)  // sin timeouts — vulnerable
}
```

### Profesional

```go
func main() {
    // Configuración
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // DI
    db := setupDB()
    repo := repository.NewPostgresProductRepository(db)
    uc := usecase.NewProductUseCase(repo, logger)
    handler := delivery.NewProductHandler(uc, logger)

    // Router con middleware
    r := chi.NewRouter()
    r.Use(requestIDMiddleware)
    r.Use(LoggingMiddleware(logger))
    r.Use(middleware.Recoverer)
    r.Route("/api/v1/products", handler.RegisterRoutes)

    // Servidor con timeouts
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      r,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Graceful shutdown
    startWithGracefulShutdown(srv, logger)
}
```
