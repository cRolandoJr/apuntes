# Concurrencia en Go

> "No comunicás compartiendo memoria. Compartís memoria comunicando." — Rob Pike

Go tiene primitivas de concurrencia nativas en el lenguaje, no como librería externa.

---

## Goroutines

Una goroutine es una función que se ejecuta concurrentemente con otras goroutines. Es mucho más ligera que un thread del SO (arranca con ~2KB de stack, crece dinámicamente).

```go
// Función normal
func doWork() {
    fmt.Println("trabajando...")
}

// La misma función como goroutine
go doWork()  // arranca concurrentemente

// Con función anónima
go func() {
    fmt.Println("goroutine anónima")
}()

// PROBLEMA CLÁSICO: main termina antes que la goroutine
func main() {
    go fmt.Println("goroutine")
    // main termina acá y la goroutine nunca ejecuta
}
```

---

## Channels — comunicación entre goroutines

Un channel permite enviar y recibir valores entre goroutines de forma segura.

```go
ch := make(chan int)       // channel sin buffer (sincrónico)
ch := make(chan int, 10)   // channel con buffer de 10 (asincrónico hasta llenar el buffer)

// Enviar
ch <- 42

// Recibir
value := <-ch

// Recibir con ok (saber si el channel está cerrado)
value, ok := <-ch
if !ok {
    fmt.Println("channel cerrado")
}
```

### Reglas de channels

| Operación | Channel nil          | Channel cerrado              | Channel normal                   |
| --------- | -------------------- | ---------------------------- | -------------------------------- |
| Enviar    | bloquea para siempre | panic                        | OK (bloquea si sin buffer lleno) |
| Recibir   | bloquea para siempre | retorna zero value, ok=false | OK (bloquea si vacío)            |
| Cerrar    | panic                | panic                        | OK                               |

```go
// Solo el EMISOR debe cerrar el channel
// Nunca cerrar un channel desde el receptor

func producer(ch chan<- int) {  // chan<- solo escritura
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)  // señal: "no hay más datos"
}

func consumer(ch <-chan int) {  // <-chan solo lectura
    for v := range ch {  // range en channel — termina cuando se cierra
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```

---

## sync.WaitGroup — esperar múltiples goroutines

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)           // incrementar antes de lanzar la goroutine
        go func(n int) {
            defer wg.Done() // decrementar cuando termina
            fmt.Printf("goroutine %d\n", n)
        }(i)
    }

    wg.Wait()  // esperar a que Done() sea llamado 5 veces
    fmt.Println("todas las goroutines terminaron")
}
```

---

## sync.Mutex — exclusión mutua para datos compartidos

Usar cuando múltiples goroutines acceden a datos compartidos y al menos una escribe.

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.RLock()    // RLock para solo lectura (permite múltiples lectores simultáneos)
    defer c.mu.RUnlock()
    return c.count
}

// sync.RWMutex — más eficiente cuando hay muchas lecturas y pocas escrituras
type Cache struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // múltiples Get pueden ejecutarse en paralelo
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // exclusivo — ningún otro Get/Set puede ejecutarse
    defer c.mu.Unlock()
    c.data[key] = value
}
```

---

## select — multiplexar channels

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() { time.Sleep(1 * time.Second); ch1 <- "uno" }()
    go func() { time.Sleep(2 * time.Second); ch2 <- "dos" }()

    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println("recibido de ch1:", msg)
        case msg := <-ch2:
            fmt.Println("recibido de ch2:", msg)
        }
    }
}

// timeout con select
select {
case result := <-workCh:
    fmt.Println("resultado:", result)
case <-time.After(5 * time.Second):
    fmt.Println("timeout: la operación tardó demasiado")
}

// non-blocking con default
select {
case msg := <-ch:
    fmt.Println("recibido:", msg)
default:
    fmt.Println("no hay mensajes disponibles ahora")
}
```

---

## Context — cancelación y timeouts propagados

`context.Context` es el mecanismo estándar para propagar cancelaciones y deadlines a través de la cadena de llamadas.

```go
// Crear contextos
ctx := context.Background()                  // raíz — usar en main() y tests
ctx := context.TODO()                        // placeholder cuando aún no sabés cuál usar
ctx, cancel := context.WithCancel(ctx)       // cancelación manual
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)  // timeout automático
ctx, cancel := context.WithDeadline(ctx, time.Now().Add(5*time.Second))
defer cancel()  // SIEMPRE llamar cancel para liberar recursos

// Pasar valores (solo metadata de request, no configuración!)
ctx = context.WithValue(ctx, keyType("requestID"), "req-abc-123")
reqID := ctx.Value(keyType("requestID")).(string)
```

### Context en toda la cadena

```go
// Handler HTTP recibe el contexto del request
func (h *ProductHandler) GetProduct(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // contexto del request HTTP
    id := chi.URLParam(r, "id")

    product, err := h.useCase.GetByID(ctx, id)  // pasa el contexto
    // ...
}

// UseCase pasa el contexto a repository
func (u *productUseCase) GetByID(ctx context.Context, id string) (*Product, error) {
    return u.repo.GetByID(ctx, id)  // pasa el contexto
}

// Repository lo usa en las queries de DB
func (r *postgresRepo) GetByID(ctx context.Context, id string) (*Product, error) {
    var p Product
    err := r.db.QueryRowContext(ctx, "SELECT ... WHERE id = $1", id).Scan(&p.ID, &p.Name)
    // Si el contexto se cancela, la query se cancela automáticamente
    return &p, err
}
```

### Escuchar cancelación

```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // context.Canceled o context.DeadlineExceeded
        default:
            // hacer trabajo
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

---

## sync.Once — ejecutar algo exactamente una vez

```go
// Patrón clásico: singleton (inicialización lazy thread-safe)
type Database struct {
    conn *sql.DB
}

var (
    dbInstance *Database
    dbOnce     sync.Once
)

func GetDB() *Database {
    dbOnce.Do(func() {
        conn, _ := sql.Open("postgres", os.Getenv("DATABASE_URL"))
        dbInstance = &Database{conn: conn}
    })
    return dbInstance
}
```

---

## Worker Pool — patrón fundamental

Limitar la cantidad de goroutines simultáneas para no saturar recursos.

```go
func ProcessItems(items []WorkItem, numWorkers int) []Result {
    jobs := make(chan WorkItem, len(items))
    results := make(chan Result, len(items))

    // Lanzar workers
    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {        // recibe trabajo hasta que jobs se cierre
                results <- process(item)    // envía resultado
            }
        }()
    }

    // Enviar trabajo
    for _, item := range items {
        jobs <- item
    }
    close(jobs)  // señal a los workers: "no hay más trabajo"

    // Esperar y cerrar results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Recolectar resultados
    var out []Result
    for r := range results {
        out = append(out, r)
    }
    return out
}
```

---

## Pipeline — transformación en cadena

```go
// Etapa 1: generar números
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Etapa 2: elevar al cuadrado
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Uso: pipeline de transformación
func main() {
    c := generate(2, 3, 4)
    out := square(c)
    for v := range out {
        fmt.Println(v) // 4, 9, 16
    }
}
```

---

## Fan-out / Fan-in (distribuir y combinar)

```go
// Fan-out: distribuir trabajo a múltiples workers
func fanOut(in <-chan Job, numWorkers int) []<-chan Result {
    channels := make([]<-chan Result, numWorkers)
    for i := 0; i < numWorkers; i++ {
        channels[i] = worker(in)  // cada worker lee del mismo canal
    }
    return channels
}

// Fan-in: combinar múltiples channels en uno
func merge(channels ...<-chan Result) <-chan Result {
    var wg sync.WaitGroup
    merged := make(chan Result)

    output := func(c <-chan Result) {
        defer wg.Done()
        for r := range c {
            merged <- r
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

---

## errgroup — manejo de errores en goroutines concurrentes

```go
import "golang.org/x/sync/errgroup"

func FetchMultiple(ctx context.Context, ids []string) ([]*Product, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]*Product, len(ids))

    for i, id := range ids {
        i, id := i, id  // captura por valor (Go < 1.22)
        g.Go(func() error {
            p, err := fetchProduct(ctx, id)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", id, err)
            }
            results[i] = p
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err  // retorna el primer error
    }
    return results, nil
}
```

---

## Práctica: Novato vs Profesional

### Novato — goroutines sin control

```go
func ProcessOrders(orders []Order) {
    for _, order := range orders {
        order := order
        go func() {  // sin límite — puede lanzar miles de goroutines
            processOrder(order)  // errores ignorados, sin cancelación
        }()
    }
    // Retorna inmediatamente sin esperar que terminen
}
```

### Profesional — goroutines controladas con context, errgroup y worker pool

```go
func ProcessOrders(ctx context.Context, orders []Order) error {
    const numWorkers = 10  // límite explícito
    jobs := make(chan Order, len(orders))

    g, ctx := errgroup.WithContext(ctx)

    // Workers
    for w := 0; w < numWorkers; w++ {
        g.Go(func() error {
            for order := range jobs {
                if err := processOrder(ctx, order); err != nil {
                    return fmt.Errorf("procesando orden %s: %w", order.ID, err)
                }
            }
            return nil
        })
    }

    // Enviar trabajo
    for _, order := range orders {
        select {
        case jobs <- order:
        case <-ctx.Done():
            close(jobs)
            return ctx.Err()
        }
    }
    close(jobs)

    return g.Wait()
}
```

---

## Detectar data races

```go
// Correr tests con detector de races
go test -race ./...

// O cualquier binario
go run -race main.go
```

El race detector es una herramienta fundamental. Siempre correrlo en CI.
