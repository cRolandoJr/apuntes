# REST API Design

---

## Principios de diseño

### Recursos, no acciones

```
// MAL — orientado a acciones (estilo RPC)
POST /getProducts
POST /createProduct
POST /deleteProduct?id=123

// BIEN — orientado a recursos (REST)
GET    /products          ← listar todos
POST   /products          ← crear uno
GET    /products/{id}     ← obtener uno
PUT    /products/{id}     ← reemplazar uno
PATCH  /products/{id}     ← actualizar parcialmente uno
DELETE /products/{id}     ← borrar uno
```

### Relaciones

```
GET  /users/{id}/orders           ← órdenes de un usuario
GET  /orders/{id}/items           ← ítems de una orden
POST /users/{id}/addresses        ← crear dirección de un usuario
```

---

## Status Codes — cuándo usar cuál

| Code                  | Significado        | Cuándo                                            |
| --------------------- | ------------------ | ------------------------------------------------- |
| 200 OK                | Éxito              | GET, PUT, PATCH exitosos                          |
| 201 Created           | Creado             | POST que crea un recurso                          |
| 204 No Content        | Sin contenido      | DELETE exitoso, PUT sin body de respuesta         |
| 400 Bad Request       | Request inválido   | Input mal formado, validación fallida             |
| 401 Unauthorized      | Sin autenticar     | No hay token o es inválido                        |
| 403 Forbidden         | Sin autorización   | Token válido pero sin permiso                     |
| 404 Not Found         | No existe          | Recurso no encontrado                             |
| 409 Conflict          | Conflicto          | El recurso ya existe, violación de unique         |
| 422 Unprocessable     | Entidad inválida   | Request bien formado pero semánticamente inválido |
| 429 Too Many Requests | Rate limit         | Demasiados requests                               |
| 500 Internal Error    | Error del servidor | Cualquier error no tipado                         |

```go
func mapDomainError(err error) (int, string) {
    var appErr *domain.AppError
    if !errors.As(err, &appErr) {
        return http.StatusInternalServerError, "error interno del servidor"
    }

    switch appErr.Code {
    case domain.ErrCodeNotFound:
        return http.StatusNotFound, appErr.Message
    case domain.ErrCodeInvalidInput:
        return http.StatusBadRequest, appErr.Message
    case domain.ErrCodeConflict:
        return http.StatusConflict, appErr.Message
    case domain.ErrCodeUnauthorized:
        return http.StatusUnauthorized, appErr.Message
    case domain.ErrCodeForbidden:
        return http.StatusForbidden, appErr.Message
    default:
        return http.StatusInternalServerError, "error interno del servidor"
    }
}
```

---

## Estructura de respuestas consistente

```go
// Response envelope — todas las respuestas con la misma forma
type Response[T any] struct {
    Data    T      `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
    Meta    *Meta  `json:"meta,omitempty"`
}

type Meta struct {
    Page       int `json:"page"`
    PageSize   int `json:"pageSize"`
    Total      int `json:"total"`
    TotalPages int `json:"totalPages"`
}

// Helpers
func Success[T any](w http.ResponseWriter, status int, data T) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(Response[T]{Data: data})
}

func Fail(w http.ResponseWriter, status int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(Response[any]{Error: message})
}

// Respuesta de lista con paginación
func SuccessList[T any](w http.ResponseWriter, data []T, meta Meta) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(Response[[]T]{Data: data, Meta: &meta})
}
```

---

## Paginación

```go
// Query params: GET /products?page=2&pageSize=20&sort=name&order=asc

type PaginationParams struct {
    Page     int    `json:"page"`
    PageSize int    `json:"pageSize"`
    Sort     string `json:"sort"`
    Order    string `json:"order"`  // "asc" | "desc"
}

func parsePagination(r *http.Request) PaginationParams {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page < 1 {
        page = 1
    }

    pageSize, _ := strconv.Atoi(r.URL.Query().Get("pageSize"))
    if pageSize < 1 || pageSize > 100 {
        pageSize = 20  // default y máximo
    }

    sort := r.URL.Query().Get("sort")
    if sort == "" {
        sort = "created_at"
    }

    order := r.URL.Query().Get("order")
    if order != "asc" && order != "desc" {
        order = "desc"
    }

    return PaginationParams{Page: page, PageSize: pageSize, Sort: sort, Order: order}
}

// En el handler
func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    params := parsePagination(r)

    products, total, err := h.useCase.GetAll(domain.GetAllInput{
        Page:     params.Page,
        PageSize: params.PageSize,
        Sort:     params.Sort,
        Order:    params.Order,
    })
    if err != nil {
        Fail(w, http.StatusInternalServerError, "error al obtener productos")
        return
    }

    totalPages := int(math.Ceil(float64(total) / float64(params.PageSize)))
    SuccessList(w, products, Meta{
        Page:       params.Page,
        PageSize:   params.PageSize,
        Total:      total,
        TotalPages: totalPages,
    })
}
```

---

## Versionado de API

```go
// Versionado en la URL — lo más común y explícito
GET /api/v1/products
GET /api/v2/products

// En el router (chi)
r.Route("/api", func(r chi.Router) {
    r.Route("/v1", func(r chi.Router) {
        r.Route("/products", v1ProductHandler.RegisterRoutes)
    })
    r.Route("/v2", func(r chi.Router) {
        r.Route("/products", v2ProductHandler.RegisterRoutes)
    })
})

// Versionado por header (más flexible, menos explícito)
// Accept: application/vnd.myapi.v2+json
```

---

## Validación de input

```go
// Validación estructurada usando go-playground/validator
import "github.com/go-playground/validator/v10"

type CreateProductRequest struct {
    Name        string  `json:"name"        validate:"required,min=2,max=255"`
    Description string  `json:"description" validate:"max=1000"`
    Price       float64 `json:"price"       validate:"required,gt=0"`
    Stock       int     `json:"stock"       validate:"min=0"`
    CategoryID  string  `json:"categoryId"  validate:"required,uuid4"`
}

var validate = validator.New()

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        Fail(w, http.StatusBadRequest, "JSON inválido")
        return
    }

    if err := validate.Struct(req); err != nil {
        // Formatear errores de validación
        var validationErrors validator.ValidationErrors
        if errors.As(err, &validationErrors) {
            fieldErrors := make(map[string]string)
            for _, e := range validationErrors {
                fieldErrors[strings.ToLower(e.Field())] = e.Tag()
            }
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusBadRequest)
            json.NewEncoder(w).Encode(map[string]any{
                "error":  "validación fallida",
                "fields": fieldErrors,
            })
            return
        }
    }

    // Convertir request a input de dominio
    input := domain.CreateProductInput{
        Name:        req.Name,
        Description: req.Description,
        Price:       req.Price,
        Stock:       req.Stock,
    }
    // ...
}
```

---

## Rate Limiting

```go
import "golang.org/x/time/rate"

// Rate limiter por IP
type ipRateLimiter struct {
    mu       sync.RWMutex
    limiters map[string]*rate.Limiter
    r        rate.Limit
    b        int
}

func newIPRateLimiter(r rate.Limit, b int) *ipRateLimiter {
    return &ipRateLimiter{
        limiters: make(map[string]*rate.Limiter),
        r:        r,
        b:        b,
    }
}

func (rl *ipRateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    if limiter, ok := rl.limiters[ip]; ok {
        return limiter
    }

    limiter := rate.NewLimiter(rl.r, rl.b)
    rl.limiters[ip] = limiter
    return limiter
}

func RateLimitMiddleware(limiter *ipRateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr
            if !limiter.getLimiter(ip).Allow() {
                w.Header().Set("Retry-After", "1")
                Fail(w, http.StatusTooManyRequests, "demasiados requests")
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Uso: 100 requests por segundo, burst de 10
limiter := newIPRateLimiter(100, 10)
r.Use(RateLimitMiddleware(limiter))
```

---

## Práctica: Novato vs Profesional

### Novato — API inconsistente

```
POST /createProduct        ← acción en la URL
POST /getProductById       ← GET semantics con POST
GET  /deleteProduct?id=1   ← delete con GET (DESTRUCTIVO)
POST /products/update      ← debería ser PUT/PATCH

Respuestas:
{ "result": "ok" }            ← sin el recurso creado
{ "error": 1 }                ← código de error, no mensaje
{ "data": [...], "ok": true } ← inconsistente según el endpoint
```

### Profesional

```
GET    /api/v1/products           → 200 { data: [...], meta: { page, total } }
POST   /api/v1/products           → 201 { data: { id, name, ... } }
GET    /api/v1/products/{id}      → 200 { data: { id, name, ... } }
PUT    /api/v1/products/{id}      → 200 { data: { id, name, ... } }
DELETE /api/v1/products/{id}      → 204 (sin body)

Errores siempre:
{ "error": "descripción del problema" }

Validación:
{ "error": "validación fallida", "fields": { "name": "required", "price": "gt" } }
```
