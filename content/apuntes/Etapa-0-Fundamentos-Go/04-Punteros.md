# Punteros

## Qué es un puntero

Un puntero es una variable que almacena la dirección de memoria de otro valor.

```
valor:    [ 42 ]  en dirección 0xc0000b4000
puntero:  [ 0xc0000b4000 ]  — apunta al valor
```

```go
x := 42
p := &x          // p es un puntero a x. & = "dame la dirección de"
fmt.Println(p)   // 0xc0000b4000 (la dirección)
fmt.Println(*p)  // 42           (* = "dame el valor en esa dirección", dereference)

*p = 100         // modificar el valor original a través del puntero
fmt.Println(x)   // 100
```

---

## El mito: "Go pasa por valor"

Go pasa **todo** por valor. Pero cuando el valor es un puntero, lo que se copia es la dirección de memoria, no el dato subyacente.

```go
type User struct {
    Name string
    Age  int
}

// Value — recibe una COPIA del struct
func DoubleAge(u User) {
    u.Age *= 2  // modifica la copia, no el original
}

// Pointer — recibe una COPIA del puntero (apunta al mismo struct)
func DoubleAgeP(u *User) {
    u.Age *= 2  // modifica el struct original
}

u := User{Name: "Rolando", Age: 26}
DoubleAge(u)
fmt.Println(u.Age)  // 26 — sin cambio

DoubleAgeP(&u)
fmt.Println(u.Age)  // 52 — cambiado
```

---

## Cuándo usar puntero vs valor

| Situación                       | Usar | Por qué                                 |
| ------------------------------- | ---- | --------------------------------------- |
| Struct grande (>64 bytes aprox) | `*T` | Evitar copiar datos grandes             |
| Método que modifica el receiver | `*T` | Para que los cambios persistan          |
| Valor opcional (puede ser nil)  | `*T` | nil significa "ausente"                 |
| Structs mutables con estado     | `*T` | Goroutines comparten la misma instancia |
| Struct pequeño, inmutable       | `T`  | Más simple, no hay nil                  |
| Primitivos (int, float, string) | `T`  | Copiar un int es igual de barato        |
| IDs, timestamps en funciones    | `T`  | No necesitan modificarse                |

### Regla de consistencia en métodos

```go
// Si UN método del tipo usa pointer receiver,
// TODOS deben usar pointer receiver para consistencia

// Bien — todos punteros
func (p *Product) SetPrice(price float64) { p.Price = price }
func (p *Product) GetName() string        { return p.Name }  // aunque no modifica
func (p *Product) IsAvailable() bool      { return p.Stock > 0 }

// Mal — mezcla de value y pointer receivers
// func (p Product) IsAvailable() bool  <- inconsistente
```

---

## nil — el puntero vacío

Un puntero no inicializado tiene valor `nil`. Desreferenciar nil causa panic.

```go
var p *User  // p == nil

// PANIC: nil pointer dereference
// fmt.Println(p.Name)

// Forma segura
if p != nil {
    fmt.Println(p.Name)
}

// Patrón común: método nil-safe
func (u *User) GetName() string {
    if u == nil {
        return ""
    }
    return u.Name
}
```

---

## new() y make()

```go
// new(T) — aloca memoria para T, retorna *T con zero value
p := new(int)     // *int apuntando a 0
u := new(User)    // *User con campos en zero value

// Equivalente manual:
var x int
p = &x

// make(T, ...) — SOLO para slices, maps y channels
// Inicializa la estructura interna (no retorna puntero)
s := make([]int, 5)          // slice de 5 enteros, len=5, cap=5
s2 := make([]int, 0, 100)    // slice vacío, cap reservada en 100
m := make(map[string]int)    // map inicializado (listo para usar)
ch := make(chan int, 10)      // channel buffered de 10
```

> Si hacés `var m map[string]int` sin `make()`, el map es nil. Asignar a un map nil causa panic.

---

## Punteros en structs — campos opcionales

```go
// * en campo indica que es opcional (puede ser nil)
type UpdateProductInput struct {
    Name        *string  // nil = "no actualizar"
    Price       *float64
    Description *string
}

// En el usecase:
func (u *productUseCase) Update(id string, input UpdateProductInput) (*Product, error) {
    product, err := u.repo.GetByID(id)
    if err != nil {
        return nil, err
    }

    if input.Name != nil {
        product.Name = *input.Name  // dereference el puntero
    }
    if input.Price != nil {
        product.Price = *input.Price
    }
    // Si es nil, no se modifica el campo

    return product, u.repo.Update(product)
}

// Helper para crear punteros a literales (Go no permite &"texto" directamente)
func ptr[T any](v T) *T { return &v }

input := UpdateProductInput{
    Name:  ptr("Nuevo nombre"),
    Price: nil,  // no actualizar precio
}
```

---

## Slices y Maps — ya son referencias

Slices y maps ya contienen internamente un puntero al backing array/map. No necesitan `*` para compartirse.

```go
// Slices — modificaciones se ven desde la función original
func AddTen(s []int) {
    for i := range s {
        s[i] += 10
    }
}

nums := []int{1, 2, 3}
AddTen(nums)
fmt.Println(nums)  // [11, 12, 13]

// PERO — append puede crear un nuevo slice si excede la capacidad
func AppendItem(s []int, item int) []int {
    return append(s, item)  // puede ser un nuevo backing array
}
// Por eso siempre retornar el slice modificado, no modificar in-place con append
```

---

## Práctica: Novato vs Profesional

### Novato — punteros donde no hacen falta / no los usa donde sí

```go
// Caso 1: puntero innecesario en entero
func GetAge(age *int) {
    fmt.Println(*age)  // por qué un puntero a int?
}

// Caso 2: retornar puntero a struct recién creado — esto en Go es OK
// (a diferencia de C/C++, Go mueve la variable al heap automáticamente)
func NewUser(name string) *User {
    return &User{Name: name}  // correcto en Go
}

// Caso 3: No manejar nil
func GetUserName(u *User) string {
    return u.Name  // PANIC si u es nil
}
```

### Profesional

```go
// Punteros solo cuando hace falta: modificación, nil-opcional, structs grandes

// Struct de dominio — pointer receiver para todo
type Product struct {
    ID    string
    Name  string
    Price float64
    Stock int
}

func (p *Product) Apply(discount float64) {
    p.Price = p.Price * (1 - discount)
}

func (p *Product) IsAvailable() bool {
    if p == nil {
        return false
    }
    return p.Stock > 0
}

// Input de update — campos opcionales con *
type UpdateInput struct {
    Name  *string
    Price *float64
}

// Helper tipado
func StringPtr(s string) *string    { return &s }
func Float64Ptr(f float64) *float64 { return &f }

// Test legible
input := UpdateInput{
    Name:  StringPtr("Nuevo nombre"),
    Price: Float64Ptr(99.99),
}
```

---

## Trucos de diagnóstico

```go
// Ver si dos punteros apuntan al mismo objeto
a := &User{Name: "A"}
b := a      // b y a apuntan al MISMO User
c := &User{Name: "A"}  // c apunta a UN NUEVO User (mismo contenido, distinta dirección)

fmt.Println(a == b)  // true  — misma dirección
fmt.Println(a == c)  // false — distintas direcciones

// Imprimir la dirección
fmt.Printf("%p\n", a)  // 0xc000014090
fmt.Printf("%p\n", b)  // 0xc000014090 (igual que a)
fmt.Printf("%p\n", c)  // 0xc0000140b0 (diferente)
```
