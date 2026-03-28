# Manejo de Errores

Go no tiene excepciones. Los errores son valores normales que se retornan explícitamente.

---

## La interfaz error

```go
// Definición en la stdlib de Go
type error interface {
    Error() string
}

// Crear un error simple
err := errors.New("algo salió mal")
err := fmt.Errorf("usuario %s no encontrado", userID)
```

---

## El patrón `(value, error)`

```go
func GetUser(id string) (*User, error) {
    if id == "" {
        return nil, errors.New("el id no puede estar vacío")
    }
    // ... buscar en DB
    if !found {
        return nil, errors.New("usuario no encontrado")
    }
    return user, nil
}

// Llamada — siempre verificar el error ANTES de usar el valor
user, err := GetUser("abc-123")
if err != nil {
    // manejar o propagar
    return fmt.Errorf("GetUser: %w", err)
}
// acá user es seguro de usar
fmt.Println(user.Name)
```

---

## Wrapping de errores — %w

Envolver errores agrega contexto sin perder el error original.

```go
// Propagar con contexto
func GetProductByID(id string) (*Product, error) {
    product, err := r.db.QueryRow(...)
    if err != nil {
        return nil, fmt.Errorf("repositorio GetProductByID id=%s: %w", id, err)
    }
    return product, nil
}

func (u *productUseCase) GetByID(id string) (*Product, error) {
    product, err := u.repo.GetProductByID(id)
    if err != nil {
        return nil, fmt.Errorf("usecase GetByID: %w", err)
    }
    return product, nil
}

// La cadena de error resultante:
// "usecase GetByID: repositorio GetProductByID id=abc: sql: no rows in result set"
```

---

## errors.Is y errors.As — inspeccionar errores envueltos

```go
// errors.Is — comparar con un error específico (incluso wrappeado)
var ErrNotFound = errors.New("not found")

err := fmt.Errorf("getUser: %w", ErrNotFound)

errors.Is(err, ErrNotFound)  // true — aunque esté envuelto

// errors.As — extraer un tipo específico de error (incluso wrappeado)
var pathErr *os.PathError

if errors.As(err, &pathErr) {
    fmt.Println("path:", pathErr.Path)
}
```

---

## Errores tipados — tipos de error de dominio

```go
// errors/domain_errors.go

type ErrorCode string

const (
    ErrCodeNotFound     ErrorCode = "NOT_FOUND"
    ErrCodeInvalidInput ErrorCode = "INVALID_INPUT"
    ErrCodeConflict     ErrorCode = "CONFLICT"
    ErrCodeUnauthorized ErrorCode = "UNAUTHORIZED"
    ErrCodeForbidden    ErrorCode = "FORBIDDEN"
)

type AppError struct {
    Code    ErrorCode
    Message string
    Cause   error  // error original (para wrapping)
}

func (e *AppError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error { return e.Cause }

// Constructores
func NewNotFoundError(msg string) *AppError {
    return &AppError{Code: ErrCodeNotFound, Message: msg}
}

func NewInvalidInputError(msg string) *AppError {
    return &AppError{Code: ErrCodeInvalidInput, Message: msg}
}

func NewConflictError(msg string) *AppError {
    return &AppError{Code: ErrCodeConflict, Message: msg}
}
```

### Usar los errores tipados

```go
// En el usecase — retornar errors de dominio
func (u *productUseCase) Create(input CreateProductInput) (*Product, error) {
    if strings.TrimSpace(input.Name) == "" {
        return nil, NewInvalidInputError("el nombre no puede estar vacío")
    }

    _, err := u.repo.GetByID(input.ID)
    if err == nil {
        return nil, NewConflictError("ya existe un producto con ese ID")
    }

    return product, nil
}

// En el delivery — mapear el error al protocolo (HTTP, GraphQL, gRPC)
func mapError(err error) (int, string) {
    var appErr *AppError
    if errors.As(err, &appErr) {
        switch appErr.Code {
        case ErrCodeNotFound:
            return 404, appErr.Message
        case ErrCodeInvalidInput:
            return 400, appErr.Message
        case ErrCodeConflict:
            return 409, appErr.Message
        case ErrCodeUnauthorized:
            return 401, appErr.Message
        }
    }
    // Error no tipado — error interno del servidor
    log.Printf("error interno: %v", err)
    return 500, "error interno del servidor"
}
```

---

## Sentinel errors — errores específicos comparables

```go
// Errores centinela — se comparan con ==  (o errors.Is sin wrapping)
var (
    ErrUserNotFound    = errors.New("usuario no encontrado")
    ErrProductNotFound = errors.New("producto no encontrado")
    ErrInvalidToken    = errors.New("token inválido o expirado")
)

// Uso en repository
func (r *memoryRepo) GetByID(id string) (*Product, error) {
    p, ok := r.products[id]
    if !ok {
        return nil, ErrProductNotFound  // sentinel error
    }
    return p, nil
}

// Uso en usecase — convertir a error de dominio
func (u *productUseCase) GetByID(id string) (*Product, error) {
    product, err := u.repo.GetByID(id)
    if err != nil {
        if errors.Is(err, ErrProductNotFound) {
            return nil, NewNotFoundError("producto no encontrado")
        }
        return nil, fmt.Errorf("obteniendo producto: %w", err)
    }
    return product, nil
}
```

---

## panic y recover — cuándo (no) usarlos

`panic` para errores de programación (bugs), NO para errores de usuario o de negocio.

```go
// OK usar panic — error de programación, falla catastrófica en init
func mustParseURL(raw string) *url.URL {
    u, err := url.Parse(raw)
    if err != nil {
        panic(fmt.Sprintf("URL inválida en configuración: %v", err))
    }
    return u
}

// NO usar panic para errores normales de negocio
// MALO:
func GetUser(id string) *User {
    user, err := db.Query(...)
    if err != nil {
        panic(err)  // mata el servidor
    }
    return user
}

// recover — solo en middleware/top-level para prevenir crashes del servidor
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic recuperado: %v\n%s", r, debug.Stack())
                http.Error(w, "error interno del servidor", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Error 1: ignorar errores con _
result, _ := strconv.Atoi(userInput)  // si falla, result=0 silenciosamente

// Error 2: mensaje sin contexto
return errors.New("error")  // ¿qué error? ¿dónde?

// Error 3: logear Y retornar (double logging)
if err != nil {
    log.Printf("Error: %v", err)   // se loguea acá
    return err                      // y también se loguea donde se maneja
}

// Error 4: panic para todo
func getConfig() Config {
    f, err := os.ReadFile("config.json")
    if err != nil {
        panic(err)  // crashea el servidor si no existe el archivo
    }
    // ...
}
```

### Profesional

```go
// 1. Nunca ignorar errores que importan
n, err := strconv.Atoi(input)
if err != nil {
    return fmt.Errorf("convirtiendo cantidad %q: %w", input, err)
}

// 2. Errores con contexto y tipo
func (u *productUseCase) Create(input CreateProductInput) (*Product, error) {
    existing, err := u.repo.GetByID(input.ID)
    if err != nil && !errors.Is(err, ErrProductNotFound) {
        return nil, fmt.Errorf("verificando existencia: %w", err)
    }
    if existing != nil {
        return nil, NewConflictError(fmt.Sprintf("producto %s ya existe", input.ID))
    }
    // ...
}

// 3. Solo logear donde se maneja (en el handler/delivery), no en las capas internas
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    product, err := h.useCase.Create(input)
    if err != nil {
        // Solo acá se loguea — el error llega con contexto completo gracias al wrapping
        h.logger.Error("crear producto", "error", err, "input", input)
        code, msg := mapError(err)
        http.Error(w, msg, code)
        return
    }
    json.NewEncoder(w).Encode(product)
}
```
