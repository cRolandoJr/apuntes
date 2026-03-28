# Slices y Maps

## Slices — el tipo de colección principal en Go

Un slice es una vista sobre un array. Tiene tres componentes: puntero al backing array, longitud (len) y capacidad (cap).

```
slice: [ ptr | len=3 | cap=5 ]
                |
                v
array: [ 1 | 2 | 3 | _ | _ ]
```

```go
// Declaración
var s []int                    // nil slice (len=0, cap=0, ptr=nil)
s := []int{}                   // slice vacío (len=0, cap=0, ptr!=nil)
s := []int{1, 2, 3}            // slice literal
s := make([]int, 5)            // len=5, cap=5, todos en 0
s := make([]int, 0, 100)       // len=0, cap=100 (reserva sin usar)
```

> Diferencia entre `nil` y vacío importa en serialización JSON: `nil` → `null`, `[]int{}` → `[]`.

---

## append

```go
s := []int{1, 2, 3}
s = append(s, 4)           // [1, 2, 3, 4]
s = append(s, 5, 6, 7)    // agregar múltiples
s = append(s, other...)    // agregar otro slice

// Siempre reasignar: append puede crear un nuevo backing array
// NUNCA hagas esto:
append(s, 4)  // el resultado se descarta — bug silencioso
```

### Cómo funciona append internamente

```go
s := make([]int, 3, 5)  // len=3, cap=5
s = append(s, 4)         // len=4, cap=5 — mismo backing array
s = append(s, 5)         // len=5, cap=5 — mismo backing array
s = append(s, 6)         // len=6, cap=10 — NUEVO backing array (cap se duplica)
```

---

## Pre-alocar capacidad — optimización clave

```go
// Novato — reinserción constante, O(n log n) de allocations
var results []Product
for _, item := range rawData {
    results = append(results, transform(item))  // puede alocar muchas veces
}

// Profesional — pre-alocar
results := make([]Product, 0, len(rawData))  // una sola alocación
for _, item := range rawData {
    results = append(results, transform(item))
}
```

---

## Slicing — crear sub-vistas

```go
s := []int{0, 1, 2, 3, 4, 5}

s[1:4]   // [1, 2, 3]       — from 1 to 4 (exclusive)
s[:3]    // [0, 1, 2]       — from 0 to 3
s[2:]    // [2, 3, 4, 5]    — from 2 to end
s[:]     // copia la referencia (misma backing array)

// Trampas de compartir backing array
a := []int{1, 2, 3, 4, 5}
b := a[1:3]  // b = [2, 3], pero comparte el mismo array que a
b[0] = 99    // modifica b Y a
fmt.Println(a)  // [1, 99, 3, 4, 5] — inesperado

// Para evitar: usar copy
b = make([]int, 2)
copy(b, a[1:3])  // copia los valores, no comparte backing array
b[0] = 99
fmt.Println(a)   // [1, 2, 3, 4, 5] — sin cambio
```

---

## copy

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)       // capacidad del destino limita cuánto se copia
n := copy(dst, src)         // n = 3 (cantidad copiada)
fmt.Println(dst)            // [1, 2, 3]

// Clonar un slice completo
clone := make([]int, len(src))
copy(clone, src)
```

---

## Operaciones comunes en slices

```go
// Eliminar elemento en índice i (sin preservar orden — O(1))
func deleteUnordered(s []int, i int) []int {
    s[i] = s[len(s)-1]  // mover el último al lugar del eliminado
    return s[:len(s)-1]
}

// Eliminar elemento en índice i (preservando orden — O(n))
func deleteOrdered(s []int, i int) []int {
    return append(s[:i], s[i+1:]...)
}

// Insertar en índice i
func insert(s []int, i int, v int) []int {
    s = append(s, 0)                // hacer espacio
    copy(s[i+1:], s[i:])           // mover elementos hacia adelante
    s[i] = v
    return s
}

// Filtrar (filter)
func filter(s []int, keep func(int) bool) []int {
    result := s[:0]  // truco: reusar el backing array sin alocar nuevo
    for _, v := range s {
        if keep(v) {
            result = append(result, v)
        }
    }
    return result
}
```

---

## Maps

```go
// Declaración — siempre inicializar con make antes de asignar
m := make(map[string]int)
m := map[string]int{"a": 1, "b": 2}  // literal

var m map[string]int  // nil map — lectura OK, escritura: PANIC
m["key"] = 1          // PANIC: assignment to entry in nil map

// Operaciones
m["key"] = 42           // asignar
v := m["key"]           // obtener (0 si no existe)
v, ok := m["key"]       // obtener con verificación
delete(m, "key")        // eliminar
len(m)                  // cantidad de pares

// Iterar (orden NO garantizado)
for k, v := range m {
    fmt.Printf("%s: %d\n", k, v)
}
// Solo keys
for k := range m {
    fmt.Println(k)
}
```

---

## Maps como sets

Go no tiene tipo `set`. Se simula con `map[T]struct{}`.

```go
// Set de strings — struct{} ocupa 0 bytes
seen := make(map[string]struct{})

seen["go"] = struct{}{}
seen["rust"] = struct{}{}

// Verificar pertenencia
if _, ok := seen["go"]; ok {
    fmt.Println("go está en el set")
}

// Eliminar duplicados de un slice
func unique(s []string) []string {
    seen := make(map[string]struct{}, len(s))
    result := make([]string, 0, len(s))
    for _, v := range s {
        if _, ok := seen[v]; !ok {
            seen[v] = struct{}{}
            result = append(result, v)
        }
    }
    return result
}
```

---

## Maps de structs — cuidado con la mutabilidad

```go
type Counter struct{ count int }

// Map con value (copia) — NO se puede modificar directamente
counters := map[string]Counter{
    "a": {count: 0},
}
// counters["a"].count++  // ERROR DE COMPILACIÓN: cannot assign to struct field in map

// Solución 1: trabajar con la copia
c := counters["a"]
c.count++
counters["a"] = c

// Solución 2: usar punteros en el map
counters := map[string]*Counter{
    "a": {count: 0},
}
counters["a"].count++  // OK — modifica el struct original
```

---

## sync.Map — map concurrente

```go
// Para uso concurrente sin mutex manual
var m sync.Map

m.Store("key", 42)

v, ok := m.Load("key")
if ok {
    fmt.Println(v.(int))
}

m.Delete("key")

m.Range(func(key, value any) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true  // retornar false para parar
})
```

> Para la mayoría de casos, un `map` + `sync.RWMutex` tiene mejor rendimiento. Usar `sync.Map` solo cuando hay muchas escrituras concurrentes en keys diferentes.

---

## Práctica: Novato vs Profesional

### Novato

```go
// 1: mapa sin inicializar
var cache map[string]*Product
cache["key"] = product  // PANIC

// 2: no verificar existencia antes de usar
product := cache["prod-001"]
fmt.Println(product.Name)  // PANIC si "prod-001" no existe

// 3: slice sin pre-alocar en hot path
var results []string
for i := 0; i < 100000; i++ {
    results = append(results, processItem(items[i]))  // muchas realocaciones
}
```

### Profesional — con el repositorio en memoria como ejemplo

```go
type memoryProductRepository struct {
    mu       sync.RWMutex
    products map[string]*domain.Product
}

func NewMemoryProductRepository() domain.ProductRepository {
    return &memoryProductRepository{
        products: make(map[string]*domain.Product),  // inicializado
    }
}

func (r *memoryProductRepository) GetAll() ([]*domain.Product, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    // Pre-alocar con la cantidad exacta
    result := make([]*domain.Product, 0, len(r.products))
    for _, p := range r.products {
        result = append(result, p)
    }
    return result, nil
}

func (r *memoryProductRepository) GetByID(id string) (*domain.Product, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    // Verificar existencia explícitamente con el pattern "ok"
    p, ok := r.products[id]
    if !ok {
        return nil, domain.ErrProductNotFound
    }
    return p, nil
}

func (r *memoryProductRepository) Create(p *domain.Product) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, ok := r.products[p.ID]; ok {
        return domain.NewConflictError("el producto ya existe")
    }
    r.products[p.ID] = p
    return nil
}
```

---

## Tablas de complejidad algorítmica

| Operación en slice              | Complejidad     |
| ------------------------------- | --------------- |
| Acceso por índice `s[i]`        | O(1)            |
| append (con capacidad)          | O(1) amortizado |
| append (sin capacidad, realoca) | O(n)            |
| copy                            | O(n)            |
| Búsqueda lineal                 | O(n)            |

| Operación en map | Complejidad   |
| ---------------- | ------------- |
| Get, Set, Delete | O(1) promedio |
| Iteración        | O(n)          |
| len()            | O(1)          |
