# Tipos y Variables

## Tipos primitivos

```go
var i int         = 42
var f float64     = 3.14
var s string      = "hola"
var b bool        = true
var by byte       = 'A'   // alias de uint8, representa bytes/ASCII
var r rune        = '💡'  // alias de int32, representa un codepoint Unicode
```

### Zero values — valores por defecto sin asignar

```go
var n int       // 0
var f float64   // 0.0
var s string    // ""
var b bool      // false
var p *int      // nil
```

> En Go no hay variables no inicializadas. Siempre tienen un valor seguro por defecto. Esto evita bugs de memoria sin inicializar que son comunes en C.

---

## Declaración de variables

```go
// Forma larga (útil cuando el tipo no se puede inferir)
var x int = 5

// Inferencia de tipo — forma más común dentro de funciones
x := 5
name := "Rolando"
price := 19.99

// Múltiples variables en una línea
a, b, c := 1, 2, 3

// Declaración sin valor inicial (usa zero value)
var count int
```

### Regla importante

`:=` solo funciona dentro de funciones. A nivel de paquete (global), usás `var`.

```go
package main

var globalConfig = "produccion"  // OK: nivel de paquete

func main() {
    localVar := 42  // OK: dentro de función
}
```

---

## Constantes e iota

```go
const Pi = 3.14159
const MaxRetries = 3
```

### iota — enums en Go

```go
type Status int

const (
    Active   Status = iota // 0
    Inactive               // 1
    Deleted                // 2
    Banned                 // 3
)

// iota con operaciones
type ByteSize float64
const (
    KB ByteSize = 1 << (10 * (iota + 1)) // 1024
    MB                                    // 1048576
    GB                                    // 1073741824
)
```

---

## Tipos numéricos — cuándo usar cuál

| Tipo      | Tamaño                        | Cuándo usarlo                               |
| --------- | ----------------------------- | ------------------------------------------- |
| `int`     | 32 o 64 bits según plataforma | Uso general, índices, contadores            |
| `int64`   | 64 bits fijo                  | IDs de DB, timestamps Unix, valores grandes |
| `int32`   | 32 bits fijo                  | Interop con C o protobuf                    |
| `uint`    | Sin signo                     | Cuando el valor nunca puede ser negativo    |
| `float64` | 64 bits                       | Precios, coordenadas, cálculos generales    |
| `float32` | 32 bits                       | Gráficos, cuando el espacio importa         |

> Para precios monetarios: nunca `float64` en producción. Usá `int` (centavos) o la librería `github.com/shopspring/decimal`.

---

## Type conversions — conversiones explícitas

Go NO hace conversiones implícitas. Hay que ser explícito:

```go
var i int = 42
var f float64 = float64(i)   // conversión explícita
var u uint = uint(f)

// String conversions
n := 65
s := string(rune(n))         // "A" — convierte a rune primero
s2 := strconv.Itoa(n)        // "65" — número a string
n2, err := strconv.Atoi("42") // string a número
```

---

## Custom types — tipos propios

```go
// Un custom type crea un tipo NUEVO aunque tenga el mismo subyacente
type UserID string
type ProductID string

var uid UserID = "abc-123"
var pid ProductID = "abc-123"

// Esto NO compila: son tipos distintos
// uid = pid  // error de compilación

// Esto SÍ compila:
uid = UserID(pid)  // conversión explícita
```

### ¿Por qué usar custom types?

Previene errores semánticos:

```go
// Sin custom types — bug silencioso
func GetUser(id string) *User { ... }
func GetOrder(id string) *Order { ... }

orderID := "order-456"
GetUser(orderID)  // compila sin error — pero es un bug

// Con custom types — el compilador lo atrapa
type UserID string
type OrderID string

func GetUser(id UserID) *User { ... }
GetUser(OrderID("order-456"))  // ERROR DE COMPILACIÓN
```

---

## Práctica: Novato vs Profesional

### Novato — válido pero impreciso

```go
func CreateOrder(userID string, productID string, quantity int) {
    // Los strings son intercambiables — nadie te avisa si los mezclás
    fmt.Println("Orden para usuario:", userID)
}

// En algún lugar del código pasan los parámetros al revés sin darse cuenta:
CreateOrder(productID, userID, 5) // compila — bug silencioso
```

### Profesional — tipado semántico

```go
type UserID string
type ProductID string

type CreateOrderInput struct {
    UserID    UserID
    ProductID ProductID
    Quantity  int
}

func CreateOrder(input CreateOrderInput) (*Order, error) {
    if input.Quantity <= 0 {
        return nil, errors.New("la cantidad debe ser mayor a cero")
    }
    // ...
}

// Ahora esto NO compila:
// CreateOrder(CreateOrderInput{UserID: ProductID("pid"), ...})
```

---

## iota para flags de bits (patrón real)

```go
type Permission uint

const (
    PermRead    Permission = 1 << iota // 001
    PermWrite                          // 010
    PermDelete                         // 100
    PermAdmin   = PermRead | PermWrite | PermDelete // 111
)

func HasPermission(userPerms, required Permission) bool {
    return userPerms&required == required
}

// Uso:
perms := PermRead | PermWrite // usuario puede leer y escribir
fmt.Println(HasPermission(perms, PermRead))   // true
fmt.Println(HasPermission(perms, PermDelete)) // false
```

---

## Puntos clave para code review

- Si ves `string` como ID de entidad (user, product, order) → preguntar si debería ser un custom type.
- Si ves `float64` para dinero → error potencial de precisión decimal.
- Si ves un `int` usado como flag de estado (0=activo, 1=inactivo, 2=borrado) → debería ser un `type Status int` con constantes `iota`.
- Si ves `var x int = 0` → redundante, el zero value ya es 0.
