# Funciones y Métodos

## Funciones — anatomía básica

```go
// Firma completa
func nombreFuncion(param1 tipo1, param2 tipo2) (tipoRetorno, error) {
    // cuerpo
    return valor, nil
}

// Ejemplo real
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("no se puede dividir por cero")
    }
    return a / b, nil
}
```

---

## Múltiples valores de retorno

Característica fundamental de Go. Se usa en todos lados.

```go
// El patrón (value, error) es idiomático en Go
func GetUser(id string) (*User, error) {
    // ...
}

// Llamada
user, err := GetUser("abc-123")
if err != nil {
    return fmt.Errorf("obteniendo usuario: %w", err)
}
```

### Retornos nombrados — usar con cuidado

```go
// Sintaxis
func ParseCoords(s string) (lat, lon float64, err error) {
    // lat, lon y err ya existen como variables locales
    parts := strings.Split(s, ",")
    if len(parts) != 2 {
        err = errors.New("formato inválido")
        return  // "naked return" — retorna lat=0, lon=0, err=error
    }
    lat, _ = strconv.ParseFloat(parts[0], 64)
    lon, _ = strconv.ParseFloat(parts[1], 64)
    return
}
```

> Los retornos nombrados son útiles para `defer` que modifica el retorno, o para documentar qué retorna cada valor. Evitá "naked returns" en funciones largas — hace difícil saber qué se retorna.

---

## Funciones variádicas

```go
func Sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

Sum(1, 2, 3)         // 6
Sum(1, 2, 3, 4, 5)   // 15

// Pasar un slice usando ...
nums := []int{1, 2, 3}
Sum(nums...)  // 6
```

---

## Funciones como valores (first-class)

En Go, las funciones son ciudadanos de primera clase: se asignan a variables, se pasan como argumentos, se retornan.

```go
// Función asignada a variable
double := func(x int) int { return x * 2 }
fmt.Println(double(5)) // 10

// Función pasada como argumento
func Apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = fn(n)
    }
    return result
}

Apply([]int{1, 2, 3}, double) // [2, 4, 6]

// Tipo de función para claridad
type Validator func(string) error

func ValidateAll(value string, validators ...Validator) error {
    for _, v := range validators {
        if err := v(value); err != nil {
            return err
        }
    }
    return nil
}
```

---

## Closures

Una closure "captura" variables del scope exterior:

```go
func MakeCounter() func() int {
    count := 0  // esta variable vive mientras exista la closure
    return func() int {
        count++
        return count
    }
}

counter := MakeCounter()
fmt.Println(counter()) // 1
fmt.Println(counter()) // 2
fmt.Println(counter()) // 3

// Caso de uso real: middleware con configuración
func WithTimeout(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### Trampa clásica con closures en goroutines

```go
// MALO — todas las goroutines capturan la misma variable i
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // puede imprimir 3, 3, 3
    }()
}

// BUENO — pasá el valor como argumento
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n) // imprime 0, 1, 2 (en algún orden)
    }(i)
}
```

---

## Métodos en Go

Un método es una función con un **receptor** (receiver). El receptor puede ser un valor o un puntero.

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Value receiver — recibe una COPIA del struct
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver — recibe el struct ORIGINAL (puede modificarlo)
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}
```

### Value receiver vs Pointer receiver

|                      | Value receiver                                | Pointer receiver                                   |
| -------------------- | --------------------------------------------- | -------------------------------------------------- |
| Modifica el original | No (trabaja en copia)                         | Sí                                                 |
| Nil safe             | Sí                                            | No (puede panic con nil)                           |
| Cuándo usar          | Structs pequeños, solo lectura                | Structs grandes, o cuando modificás                |
| Interfaz             | Solo satisface interfaces con value receivers | Satisface interfaces con value Y pointer receivers |

```go
// Regla práctica: si UN método del tipo necesita pointer receiver,
// usá pointer receiver en TODOS para consistencia

type User struct {
    Name  string
    Email string
    Age   int
}

func (u *User) Greet() string {       // pointer — consistencia
    return "Hola, " + u.Name
}

func (u *User) Birthday() {           // pointer — modifica
    u.Age++
}

func (u *User) IsAdult() bool {       // podría ser value, pero es pointer por consistencia
    return u.Age >= 18
}
```

---

## Functional Options Pattern

Patrón profesional para configurar structs sin constructores con 10 parámetros:

```go
// Novato — constructor con muchos parámetros (frágil, orden importa)
func NewServer(host string, port int, timeout time.Duration, maxConns int, debug bool) *Server {
    // ...
}

// Llamada confusa:
NewServer("localhost", 8080, 30*time.Second, 100, false)

// =====================

// Profesional — Functional Options
type Server struct {
    host     string
    port     int
    timeout  time.Duration
    maxConns int
    debug    bool
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func WithDebug() Option {
    return func(s *Server) {
        s.debug = true
    }
}

func NewServer(opts ...Option) *Server {
    // Valores por defecto sensatos
    s := &Server{
        host:     "localhost",
        port:     8080,
        timeout:  30 * time.Second,
        maxConns: 100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Llamada legible, extensible, con defaults
srv := NewServer(
    WithPort(9090),
    WithDebug(),
    WithTimeout(60*time.Second),
)
```

---

## defer

Ejecuta una función DESPUÉS de que la función actual retorna. Se ejecuta LIFO (último en entrar, primero en salir).

```go
func ReadFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // se garantiza el cierre aunque haya panic o error

    return io.ReadAll(f)
}

// Múltiples defers — se ejecutan en orden LIFO
func example() {
    defer fmt.Println("tercero")
    defer fmt.Println("segundo")
    defer fmt.Println("primero")
    // Output: primero, segundo, tercero
}
```

### defer con retorno nombrado — patrón avanzado

```go
func DoWork() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recuperado: %v", r)
        }
    }()
    // ...código que podría hacer panic...
    return nil
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Función con demasiados parámetros, sin errores claros
func CreateProduct(db *sql.DB, name string, price float64, stock int, categoryID int,
                   active bool, createdBy string, tags []string) error {
    // 200 líneas de código mezclando validación, lógica y DB
    if name == "" {
        return errors.New("error")  // error poco descriptivo
    }
    // SQL directo en la función
    db.Exec("INSERT INTO products ...", name, price, stock)
    return nil
}
```

### Profesional

```go
// Input tipado con validación separada
type CreateProductInput struct {
    Name       string
    Price      float64
    Stock      int
    CategoryID ProductCategoryID
    Tags       []string
}

func (i CreateProductInput) Validate() error {
    if strings.TrimSpace(i.Name) == "" {
        return domain.NewInvalidInputError("el nombre no puede estar vacío")
    }
    if i.Price <= 0 {
        return domain.NewInvalidInputError("el precio debe ser mayor a cero")
    }
    if i.Stock < 0 {
        return domain.NewInvalidInputError("el stock no puede ser negativo")
    }
    return nil
}

// UseCase limpio — no sabe nada de SQL ni HTTP
func (u *productUseCase) Create(ctx context.Context, input CreateProductInput) (*Product, error) {
    if err := input.Validate(); err != nil {
        return nil, err
    }

    product := &Product{
        ID:         NewProductID(),
        Name:       input.Name,
        Price:      input.Price,
        Stock:      input.Stock,
        CategoryID: input.CategoryID,
        Tags:       input.Tags,
        CreatedAt:  time.Now(),
    }

    if err := u.repo.Create(ctx, product); err != nil {
        return nil, fmt.Errorf("creando producto: %w", err)
    }

    return product, nil
}
```
