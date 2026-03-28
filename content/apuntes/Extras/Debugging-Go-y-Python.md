# Debugging — Go y Python

Saber debuggear es la diferencia entre tardar 5 minutos o 2 horas en encontrar un bug. No uses `fmt.Println` o `print()` como única herramienta.

---

## Debugging en Go con `dlv` (Delve)

### Instalación

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

### Ejecutar el programa bajo el debugger

```bash
# Caso más común: debuggear la app
dlv debug ./cmd/main.go

# Si el programa recibe argumentos:
dlv debug ./cmd/main.go -- --port 8080 --env dev

# Debuggear tests
dlv test ./internal/usecase/... -- -run TestCreateProduct

# Adjuntarse a un proceso que ya está corriendo
dlv attach <PID>
```

### Comandos del REPL de Delve

```
(dlv) break main.go:42                    ← breakpoint en línea 42
(dlv) break handler.ProductHandler.Create ← breakpoint en función
(dlv) breakpoints                         ← listar breakpoints
(dlv) clear 1                             ← eliminar breakpoint 1

(dlv) continue   (c)    ← correr hasta el próximo breakpoint
(dlv) next       (n)    ← siguiente línea (sin entrar a funciones)
(dlv) step       (s)    ← siguiente línea (entra a funciones)
(dlv) stepout           ← salir de la función actual

(dlv) print <variable>  (p) ← imprimir valor
(dlv) locals            ← ver todas las variables locales
(dlv) args              ← ver argumentos de la función actual
(dlv) vars              ← ver variables del paquete

(dlv) goroutines        ← listar goroutines activas
(dlv) goroutine 5       ← cambiar a goroutine 5
(dlv) stack             ← ver el call stack

(dlv) quit   (q)        ← salir
```

### Breakpoints condicionales

```
(dlv) break product_usecase.go:45 if product.Price < 0
(dlv) break repo.go:78 if id == 9999
```

### Debugging en VS Code (Go)

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug API",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/main.go",
      "env": {
        "DATABASE_URL": "postgresql://postgres:password@localhost:5432/dev_db",
        "SECRET_KEY": "dev-secret",
      },
      "args": [],
    },
    {
      "name": "Debug Tests",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}/internal/usecase",
      "args": ["-run", "TestCreateProduct", "-v"],
    },
  ],
}
```

Con esta config podés poner breakpoints directo en el código y presionar F5.

---

## Técnicas de debugging en Go

### Imprimir estructuras completas

```go
import "fmt"

// %v = valores, %+v = nombres + valores, %#v = sintaxis Go
fmt.Printf("%+v\n", product)
// {ID:1 Name:Laptop Price:1500 Stock:10}

fmt.Printf("%#v\n", product)
// domain.Product{ID:1, Name:"Laptop", Price:1500, Stock:10}
```

### Leer variables de goroutines concurrentes

```go
// Agregar identificador a cada goroutine para debuggear race conditions
import "runtime"

func goroutineID() int {
    var buf [64]byte
    n := runtime.Stack(buf[:], false)
    // "goroutine 42 [running]:\n..."
    // parsear el número
    id := 0
    fmt.Sscanf(string(buf[:n]), "goroutine %d", &id)
    return id
}
```

### Race detector — encontrar data races

```bash
# Correr con el race detector (más lento pero detecta problemas de concurrencia)
go run -race ./cmd/main.go
go test -race ./...
```

### `pprof` — profiling de performance

```go
// main.go — agregar para poder profilear en producción
import (
    _ "net/http/pprof"  // registra endpoints /debug/pprof/
    "net/http"
)

go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
# Capturar CPU profile por 30 segundos
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Capturar heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Visualizar (requiere graphviz)
(pprof) web
(pprof) top10
```

---

## Debugging en Python

### `pdb` — el debugger de la stdlib

```python
# Opción 1: insertar breakpoint en el código
import pdb; pdb.set_trace()  # Python < 3.7
breakpoint()                  # Python 3.7+ — equivalente, más limpio

# El programa se pausa en esa línea y abre el REPL
```

```
(Pdb) l              ← listar código alrededor de la línea actual
(Pdb) n              ← next (siguiente línea, sin entrar a funciones)
(Pdb) s              ← step (siguiente línea, entra a funciones)
(Pdb) c              ← continue (correr hasta el próximo breakpoint)
(Pdb) r              ← return (correr hasta retornar de la función actual)
(Pdb) q              ← quit

(Pdb) p variable     ← imprimir valor
(Pdb) pp variable    ← pretty print
(Pdb) p product.__dict__   ← ver todos los atributos de un objeto

(Pdb) where  (w)     ← ver call stack
(Pdb) up     (u)     ← subir un frame en el call stack
(Pdb) down   (d)     ← bajar un frame

(Pdb) b 45           ← breakpoint en línea 45
(Pdb) b product.py:78   ← breakpoint en archivo:línea
(Pdb) b create_product  ← breakpoint en función
(Pdb) condition 1 price < 0  ← breakpoint condicional

# Ejecutar código Python en el contexto actual
(Pdb) !product.name = "debug"
(Pdb) !print([p.id for p in products])
```

### `ipdb` — pdb mejorado (más legible)

```bash
uv add --dev ipdb
```

```python
import ipdb; ipdb.set_trace()  # igual que breakpoint() pero con colores y tab completion
```

### Post-mortem debugging — analizar un crash después de que ocurrió

```python
# python -m pdb mi_script.py — correr en modo debug automáticamente
python -m pdb scripts/cleanup.py

# Post-mortem desde código — útil en tests
import pdb
try:
    result = crashy_function()
except Exception:
    pdb.post_mortem()   # entra al REPL en el estado donde ocurrió el error
```

### Debugging en VS Code (Python)

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI Debug",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["src.mi_proyecto.main:app", "--reload", "--port", "8000"],
      "env": {
        "DATABASE_URL": "postgresql+asyncpg://postgres:password@localhost:5432/dev_db",
      },
      "jinja": true,
    },
    {
      "name": "Debug Tests",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["-xvs", "tests/"],
      "console": "integratedTerminal",
    },
    {
      "name": "Debug Script",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}", // debuggea el archivo activo
      "console": "integratedTerminal",
    },
  ],
}
```

Con esto: F9 → poner breakpoints, F5 → iniciar el debug, F10 → next, F11 → step into.

---

## Técnicas de debugging en Python

### Inspeccionar objetos desconocidos

```python
# Qué tipo es
type(obj)
type(obj).__name__

# Qué atributos tiene
dir(obj)
vars(obj)          # solo __dict__ (atributos de instancia)

# Documentación
help(obj)
help(type(obj))

# Para objetos Pydantic
product.model_fields    # campos del modelo
product.model_dump()    # como dict — más legible
```

### `rich` para debug output bonito

```bash
uv add --dev rich
```

```python
from rich import print as rprint
from rich.pretty import pprint
from rich.traceback import install

install()  # tracebacks de Python más legibles automáticamente

# Print con colores y estructura
rprint({"key": "value", "nested": {"a": 1, "b": [1, 2, 3]}})
pprint(product.model_dump())   # con indentación y colores
```

### Logging como herramienta de debug

```python
import logging

# Activar DEBUG temporalmente
logging.basicConfig(level=logging.DEBUG)

# O solo para ciertos módulos
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)  # ver SQL generado
logging.getLogger("httpx").setLevel(logging.DEBUG)             # ver requests HTTP
```

---

## Debugging de queries SQL

### Go — loguear queries de pgx

```go
// En la configuración del pool de pgx
config.ConnConfig.Tracer = &pgxLogger{log: logger}

type pgxLogger struct {
    log *slog.Logger
}

func (l *pgxLogger) TraceQueryStart(ctx context.Context, _ *pgx.Conn, data pgx.TraceQueryStartData) context.Context {
    l.log.Debug("SQL query", "sql", data.SQL, "args", data.Args)
    return ctx
}
```

### Python — loguear SQL de SQLAlchemy

```python
# database.py — solo en desarrollo
engine = create_async_engine(
    settings.database_url,
    echo=settings.debug,  # True en dev: loguea todo el SQL
)

# O más granular:
import logging
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
logging.getLogger("sqlalchemy.pool").setLevel(logging.DEBUG)  # info del pool
```

---

## Checklist cuando algo falla

```
1. Leer el error completo — incluyendo el stack trace desde arriba
2. Buscar la línea de TU código en el traceback (no la de la librería)
3. ¿Qué valores tiene las variables en ese punto? → print/breakpoint
4. ¿Funciona en aislamiento? → test unitario pequeño
5. ¿Cuándo empezó a fallar? → git log, git bisect
6. ¿Hay un test que capture este caso? → si no, escribirlo primero
7. ¿El error habla de tipos? → revisar type hints, model_dump, vars()
8. ¿Es un problema async? → ¿estás usando await donde hace falta?
9. Para crashes en producción: buscar en los logs con el timestamp del error
```
