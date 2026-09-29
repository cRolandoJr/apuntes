# Fase 3: Programación & Automatización — Meses 7 a 9

> **Objetivo**: Dominar Go como lenguaje principal, Python como herramienta de scripting, entender bases de datos y APIs REST. Un Platform Engineer que no sabe programar es un SysAdmin con herramientas modernas — la diferencia es el código.

---

## Mes 7: Go en Profundidad

Ya tocaste Go con tu proyecto. Ahora vamos a entender **de verdad** cada concepto.

### 7.1 El Sistema de Tipos de Go

**Tipos básicos**:

```go
// Enteros
var a int       // Tamaño depende de la plataforma (64-bit en x86_64)
var b int8      // -128 a 127
var c int16     // -32768 a 32767
var d int32     // -2 mil millones a 2 mil millones
var e int64     // Muy grande
var f uint      // Solo positivos (unsigned)
var g uint8     // 0 a 255 (= byte)

// Punto flotante
var h float32   // 7 dígitos de precisión
var i float64   // 15 dígitos de precisión (USAR ESTE por defecto)

// String
var j string    // UTF-8 por defecto. Inmutable.

// Bool
var k bool      // true o false

// Byte y Rune
var l byte      // Alias de uint8 (un byte)
var m rune      // Alias de int32 (un carácter Unicode)
```

**Zero values**: En Go, toda variable tiene un valor por defecto:

```go
var i int       // 0
var f float64   // 0.0
var s string    // "" (string vacío)
var b bool      // false
var p *int      // nil (puntero nulo)
var sl []int    // nil (slice nulo)
var mp map[string]int  // nil (map nulo — NO se puede usar sin make())
```

### 7.2 Structs y Métodos

```go
// Un struct agrupa datos relacionados
type Server struct {
    Name     string
    IP       string
    Port     int
    IsActive bool
}

// Crear instancias
s1 := Server{Name: "web-01", IP: "10.0.1.10", Port: 80, IsActive: true}
s2 := Server{Name: "db-01", IP: "10.0.1.20", Port: 5432}  // IsActive = false (zero value)

// Acceder a campos
fmt.Println(s1.Name)  // "web-01"

// Métodos: funciones asociadas a un struct
func (s *Server) Shutdown() {
    // (s *Server) es el "receiver" — este método pertenece a *Server
    // Usamos puntero (*Server) porque vamos a MODIFICAR el struct
    s.IsActive = false
    fmt.Printf("Servidor %s apagado\n", s.Name)
}

func (s Server) Address() string {
    // Sin puntero (Server) porque solo LEEMOS, no modificamos
    return fmt.Sprintf("%s:%d", s.IP, s.Port)
}

// Uso
s1.Shutdown()              // s1.IsActive ahora es false
addr := s1.Address()       // "10.0.1.10:80"
```

**¿Cuándo usar puntero en el receiver?**

- `*Server` (puntero): Cuando el método **modifica** el struct, o cuando el struct es grande (evitar copiar)
- `Server` (valor): Cuando solo **lee** datos. Pero en la práctica, si algún método usa puntero, usá puntero en TODOS (consistencia)

### 7.3 Interfaces — El Concepto Más Importante de Go

```go
// Una interfaz define COMPORTAMIENTO, no datos
type Repository interface {
    Save(item Item) error
    FindByID(id string) (Item, error)
    Delete(id string) error
}

// CUALQUIER struct que implemente estos 3 métodos SATISFACE la interfaz
// No hay "implements" explícito — es implícito (duck typing)

type MemoryRepository struct {
    data map[string]Item
}

func (r *MemoryRepository) Save(item Item) error { /* ... */ return nil }
func (r *MemoryRepository) FindByID(id string) (Item, error) { /* ... */ return Item{}, nil }
func (r *MemoryRepository) Delete(id string) error { /* ... */ return nil }
// ¡MemoryRepository implementa Repository automáticamente!

type PostgresRepository struct {
    db *sql.DB
}

func (r *PostgresRepository) Save(item Item) error { /* SQL INSERT */ return nil }
func (r *PostgresRepository) FindByID(id string) (Item, error) { /* SQL SELECT */ return Item{}, nil }
func (r *PostgresRepository) Delete(id string) error { /* SQL DELETE */ return nil }
// PostgresRepository TAMBIÉN implementa Repository

// La función que USA la interfaz no sabe (ni le importa) cuál implementación tiene:
func ProcessItems(repo Repository) {
    // Puede ser Memory, Postgres, MongoDB, un Mock de test... da igual
    item, _ := repo.FindByID("123")
    fmt.Println(item)
}
```

**Interfaces de la biblioteca estándar que usarás constantemente**:

```go
// io.Reader — Lee bytes de alguna fuente
type Reader interface {
    Read(p []byte) (n int, err error)
}
// Lo implementa: archivos, conexiones TCP, cuerpos HTTP, buffers...

// io.Writer — Escribe bytes a algún destino
type Writer interface {
    Write(p []byte) (n int, err error)
}
// Lo implementa: archivos, conexiones TCP, stdout, buffers...

// error — El manejo de errores de Go
type error interface {
    Error() string
}
// Cualquier struct con un método Error() string es un error

// fmt.Stringer — Representación en string de un tipo
type Stringer interface {
    String() string
}
```

### 7.4 Goroutines y Channels

**Goroutine**: Un hilo de ejecución ultra-liviano. Cada servidor HTTP de Go maneja cada request en su propia goroutine.

```go
// Lanzar una goroutine: go + llamada a función
go hacerAlgo()                    // Se ejecuta en paralelo
go func() { fmt.Println("hola") }()  // Goroutine anónima

// Problema: ¿cómo comunicar goroutines entre sí?
// Respuesta: CHANNELS
```

**Channel**: Un tubo tipado para pasar datos entre goroutines.

```go
// Crear un channel
ch := make(chan string)       // Channel de strings, sin buffer
ch := make(chan string, 10)   // Channel con buffer de 10 elementos

// Enviar datos al channel
ch <- "hola"                  // Bloquea hasta que alguien lea (si no hay buffer)

// Recibir datos del channel
msg := <-ch                   // Bloquea hasta que alguien envíe

// Ejemplo práctico: verificar salud de múltiples servidores en paralelo
func checkHealth(servers []string) map[string]bool {
    results := make(chan struct {
        server string
        alive  bool
    }, len(servers))

    for _, srv := range servers {
        go func(s string) {
            // Cada servidor se verifica en su propia goroutine
            alive := ping(s)
            results <- struct {
                server string
                alive  bool
            }{s, alive}
        }(srv)
    }

    health := make(map[string]bool)
    for range servers {
        r := <-results
        health[r.server] = r.alive
    }
    return health
}
```

**sync.WaitGroup**: Esperar a que varias goroutines terminen.

```go
var wg sync.WaitGroup

for _, server := range servers {
    wg.Add(1)  // "Esperá a uno más"
    go func(s string) {
        defer wg.Done()  // "Ya terminé"
        processServer(s)
    }(server)
}

wg.Wait()  // Bloquea hasta que TODAS llamaron Done()
fmt.Println("Todos los servidores procesados")
```

**sync.Mutex**: Lo que ya usaste en tu proyecto. Proteger datos compartidos.

```go
var (
    mu      sync.Mutex
    counter int
)

// Sin mutex: race condition (counter se corrompe)
// Con mutex: solo una goroutine modifica counter a la vez
func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

### 7.5 Manejo de Errores en Go (Patrón Completo)

```go
// Go NO tiene excepciones. Los errores son VALORES que se retornan.

// 1. Error simple
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("no se puede dividir por cero")
    }
    return a / b, nil
}

result, err := divide(10, 0)
if err != nil {
    log.Printf("Error: %v", err)
    return  // Manejar y salir
}
fmt.Println(result)

// 2. Errores tipados (como tu AppError)
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s con ID %s no encontrado", e.Resource, e.ID)
}

// 3. Verificar tipo de error
var nfe *NotFoundError
if errors.As(err, &nfe) {
    // Es un NotFoundError — devolver 404
    fmt.Printf("No encontrado: %s %s\n", nfe.Resource, nfe.ID)
}

// 4. Wrapping: agregar contexto a un error
func getUser(id string) (*User, error) {
    user, err := db.FindByID(id)
    if err != nil {
        return nil, fmt.Errorf("getUser(%s): %w", id, err)
        // %w = wrap — envuelve el error original
    }
    return user, nil
}
// Ahora errors.As(err, &nfe) sigue funcionando a través del wrapping
```

### 7.6 Paquetes y Módulos

```go
// go.mod — Define tu módulo y sus dependencias
module gestion_productos    // Nombre del módulo
go 1.25.5                   // Versión mínima de Go

require (
    github.com/99designs/gqlgen v0.17.88  // Dependencia + versión
    github.com/google/uuid v1.6.0
)

// Importar paquetes
import (
    "fmt"                    // Paquete estándar
    "net/http"               // Paquete estándar anidado
    "gestion_productos/internal/domain"  // Paquete de tu módulo
    "github.com/google/uuid"             // Paquete externo
)

// Regla de visibilidad: Mayúscula = público, minúscula = privado
type Product struct { ... }   // Público — visible fuera del paquete
type productUseCase struct { ... }  // Privado — solo dentro del paquete usecase
func NewProductUseCase() { ... }    // Público — constructor accesible
func validate(p Product) { ... }    // Privado — función interna

// Comandos de módulos
go mod init gestion_productos  // Crear módulo
go mod tidy                    // Limpiar dependencias
go get github.com/lib/pq       // Agregar dependencia
go list -m all                  // Ver todas las dependencias
```

**Ejercicios verificables**:

```bash
# Crear un programa que use goroutines para escanear puertos
mkdir -p /tmp/go-practice && cd /tmp/go-practice
go mod init port-scanner

cat << 'EOF' > main.go
package main

import (
    "fmt"
    "net"
    "os"
    "sync"
    "time"
)

func scanPort(host string, port int, wg *sync.WaitGroup, results chan<- int) {
    defer wg.Done()
    addr := fmt.Sprintf("%s:%d", host, port)
    conn, err := net.DialTimeout("tcp", addr, 500*time.Millisecond)
    if err != nil {
        return
    }
    conn.Close()
    results <- port
}

func main() {
    host := "localhost"
    if len(os.Args) > 1 {
        host = os.Args[1]
    }

    var wg sync.WaitGroup
    results := make(chan int, 100)

    for port := 1; port <= 1024; port++ {
        wg.Add(1)
        go scanPort(host, port, &wg, results)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    fmt.Printf("Puertos abiertos en %s:\n", host)
    for port := range results {
        fmt.Printf("  %d/tcp abierto\n", port)
    }
}
EOF

go run main.go localhost
# Deberías ver los puertos abiertos en tu máquina
```

---

## Mes 8: Python para Automatización + Bases de Datos

### 8.1 Python — Lo Que Necesita un Platform Engineer

No necesitás ser un experto en Python, pero sí usarlo como herramienta. Scripts de automatización, parseo de datos, interacción con APIs.

```python
#!/usr/bin/env python3
"""Script de automatización: verificar servidores"""

import subprocess
import json
import sys
from datetime import datetime
from pathlib import Path

def run_command(cmd: str) -> tuple[str, int]:
    """Ejecutar un comando y devolver output + exit code."""
    result = subprocess.run(
        cmd, shell=True, capture_output=True, text=True
    )
    return result.stdout.strip(), result.returncode

def check_disk_usage(threshold: int = 80) -> list[dict]:
    """Verificar uso de disco, retornar particiones por encima del umbral."""
    output, _ = run_command("df -h --output=source,pcent,target | tail -n +2")
    alerts = []
    for line in output.splitlines():
        parts = line.split()
        if len(parts) >= 3:
            usage = int(parts[1].rstrip('%'))
            if usage >= threshold:
                alerts.append({
                    "device": parts[0],
                    "usage": usage,
                    "mount": parts[2],
                })
    return alerts

def check_service(name: str) -> bool:
    """Verificar si un servicio systemd está activo."""
    _, code = run_command(f"systemctl is-active --quiet {name}")
    return code == 0

def main():
    report = {
        "timestamp": datetime.now().isoformat(),
        "hostname": subprocess.getoutput("hostname"),
        "checks": {}
    }

    # Verificar disco
    disk_alerts = check_disk_usage(80)
    report["checks"]["disk"] = {
        "status": "warning" if disk_alerts else "ok",
        "alerts": disk_alerts,
    }

    # Verificar servicios
    services = ["sshd", "docker", "nginx"]
    for svc in services:
        active = check_service(svc)
        report["checks"][svc] = {
            "status": "ok" if active else "critical",
        }

    # Guardar reporte
    output_path = Path(f"/tmp/report-{datetime.now():%Y%m%d-%H%M%S}.json")
    output_path.write_text(json.dumps(report, indent=2))
    print(json.dumps(report, indent=2))

    # Exit code basado en si hay problemas
    has_critical = any(
        c["status"] == "critical"
        for c in report["checks"].values()
    )
    sys.exit(1 if has_critical else 0)

if __name__ == "__main__":
    main()
```

**Librerías Python esenciales para Platform Engineering**:

```python
# Requests — HTTP client (pip install requests)
import requests
response = requests.get("https://api.github.com/users/octocat")
data = response.json()
print(data["login"])

# PyYAML — Parsear YAML (configs de K8s, Ansible, etc.)
import yaml
with open("docker-compose.yml") as f:
    config = yaml.safe_load(f)
print(config["services"]["api"]["ports"])

# Jinja2 — Templates (generar configs dinámicamente)
from jinja2 import Template
template = Template("""
server {
    listen {{ port }};
    server_name {{ domain }};
    location / {
        proxy_pass http://{{ backend }};
    }
}
""")
config = template.render(port=80, domain="api.local", backend="localhost:8081")

# boto3 — AWS SDK (pip install boto3)
import boto3
ec2 = boto3.client("ec2")
instances = ec2.describe_instances()

# Paramiko — SSH desde Python (pip install paramiko)
import paramiko
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect("servidor", username="deploy", key_filename="~/.ssh/id_ed25519")
stdin, stdout, stderr = ssh.exec_command("uptime")
print(stdout.read().decode())
ssh.close()
```

### 8.2 Bases de Datos — SQL Fundamental

Tu proyecto usa almacenamiento en memoria. En la vida real, usarás PostgreSQL (u otra BD relacional).

**Conceptos fundamentales**:

```sql
-- Una tabla es como un struct con múltiples instancias
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    price       DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- CRUD básico
-- Create
INSERT INTO products (name, price, stock)
VALUES ('Laptop', 999.99, 10)
RETURNING *;

-- Read
SELECT * FROM products WHERE id = 'some-uuid';
SELECT * FROM products WHERE price > 500 ORDER BY name;
SELECT * FROM products LIMIT 10 OFFSET 20;  -- Paginación

-- Update
UPDATE products SET price = 899.99 WHERE id = 'some-uuid';

-- Delete
DELETE FROM products WHERE id = 'some-uuid';
```

**Índices**: Sin índices, cada SELECT recorre TODA la tabla. Con índice, es una búsqueda directa.

```sql
-- Índice en name para búsquedas rápidas
CREATE INDEX idx_products_name ON products (name);

-- Índice compuesto
CREATE INDEX idx_products_price_stock ON products (price, stock);

-- Ver el plan de ejecución de una query
EXPLAIN ANALYZE SELECT * FROM products WHERE name = 'Laptop';
-- Si dice "Seq Scan" → recorre toda la tabla (lento)
-- Si dice "Index Scan" → usa el índice (rápido)
```

**Relaciones**:

```sql
-- Ejemplo: Productos pertenecen a Categorías
CREATE TABLE categories (
    id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    -- FOREIGN KEY: apunta a otra tabla. Garantiza integridad referencial.
    -- ON DELETE SET NULL: si se borra la categoría, el producto queda sin categoría
    -- ON DELETE CASCADE: si se borra la categoría, se borra el producto
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- JOIN: combinar datos de ambas tablas
SELECT p.name, p.price, c.name AS category
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.price > 100;
```

**Migraciones**: Cambios al esquema de la BD versionados como código.

```sql
-- migrations/001_create_products.up.sql
CREATE TABLE products (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(255) NOT NULL,
    price      DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock      INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- migrations/001_create_products.down.sql
DROP TABLE IF EXISTS products;

-- migrations/002_add_description.up.sql
ALTER TABLE products ADD COLUMN description TEXT;

-- migrations/002_add_description.down.sql
ALTER TABLE products DROP COLUMN description;
```

Herramientas de migración: `golang-migrate`, `goose`, `atlas`. Se ejecutan en orden numérico y trackean cuáles ya se aplicaron.

**Ejercicio verificable (con Docker)**:

```bash
# Levantar PostgreSQL con Docker
docker run -d --name pg-lab \
  -e POSTGRES_USER=rolando \
  -e POSTGRES_PASSWORD=lab123 \
  -e POSTGRES_DB=productos \
  -p 5432:5432 \
  postgres:16

# Conectarse
docker exec -it pg-lab psql -U rolando -d productos

# Dentro de psql, ejecutá los CREATE TABLE, INSERT, SELECT de arriba
# Verificá: \dt (listar tablas), \d products (describir tabla)
# Salir: \q

# Limpiar
docker stop pg-lab && docker rm pg-lab
```

---

## Mes 9: APIs REST y Go HTTP Server

### 9.1 REST desde Go (sin frameworks)

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type Server struct {
    Name   string `json:"name"`
    Status string `json:"status"`
}

func handleServers(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        servers := []Server{
            {Name: "web-01", Status: "running"},
            {Name: "db-01", Status: "running"},
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(servers)

    case http.MethodPost:
        var server Server
        if err := json.NewDecoder(r.Body).Decode(&server); err != nil {
            http.Error(w, "JSON inválido", http.StatusBadRequest)
            return
        }
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(server)

    default:
        http.Error(w, "Método no permitido", http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/api/servers", handleServers)
    log.Println("Servidor en :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 9.2 Testing en Go (En Profundidad)

```go
// product_usecase_test.go
package usecase_test

import (
    "testing"
    "gestion_productos/internal/domain"
    "gestion_productos/internal/usecase"
    "gestion_productos/internal/repository"
)

// Table-driven tests: el patrón estándar de Go
func TestCreateProduct_Validation(t *testing.T) {
    repo := repository.NewMemoryProductRepository()
    uc := usecase.NewProductUseCase(repo)

    tests := []struct {
        name    string              // Nombre del sub-test
        input   domain.CreateProductInput
        wantErr bool                // ¿Esperamos error?
        errCode domain.ErrorCode    // ¿Qué código de error?
    }{
        {
            name:    "valid product",
            input:   domain.CreateProductInput{Name: "Laptop", Price: 999.99, Stock: 10},
            wantErr: false,
        },
        {
            name:    "empty name",
            input:   domain.CreateProductInput{Name: "", Price: 100, Stock: 5},
            wantErr: true,
            errCode: domain.ErrCodeInvalidInput,
        },
        {
            name:    "negative price",
            input:   domain.CreateProductInput{Name: "Test", Price: -10, Stock: 5},
            wantErr: true,
            errCode: domain.ErrCodeInvalidInput,
        },
        {
            name:    "negative stock",
            input:   domain.CreateProductInput{Name: "Test", Price: 100, Stock: -1},
            wantErr: true,
            errCode: domain.ErrCodeInvalidInput,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            product, err := uc.Create(tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("esperaba error pero no hubo")
                }
                var appErr *domain.AppError
                if !errors.As(err, &appErr) {
                    t.Fatalf("error no es AppError: %v", err)
                }
                if appErr.Code != tt.errCode {
                    t.Errorf("código: got %s, want %s", appErr.Code, tt.errCode)
                }
            } else {
                if err != nil {
                    t.Fatalf("no esperaba error: %v", err)
                }
                if product.Name != tt.input.Name {
                    t.Errorf("nombre: got %s, want %s", product.Name, tt.input.Name)
                }
            }
        })
    }
}

// Benchmarks: medir rendimiento
func BenchmarkCreateProduct(b *testing.B) {
    repo := repository.NewMemoryProductRepository()
    uc := usecase.NewProductUseCase(repo)
    input := domain.CreateProductInput{Name: "Bench", Price: 100, Stock: 5}

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        uc.Create(input)
    }
}
```

```bash
# Ejecutar tests
go test ./...                      # Todos los tests
go test -v ./internal/usecase/     # Verbose, solo usecase
go test -run TestCreate ./...      # Solo tests que matcheen "TestCreate"
go test -cover ./...               # Con cobertura
go test -bench=. ./...             # Benchmarks
go test -race ./...                # Detector de race conditions
```

---

## Proyecto Integrador de Fase 3

### "API de Inventario de Servidores"

Construí una API REST en Go **desde cero** (sin copiar tu proyecto existente):

1. **Domain**: Struct `Server` con: ID, Hostname, IP, OS, CPU, RAM, Status, CreatedAt
2. **Repository**: Implementá DOS repositorios:
   - `MemoryRepository` (map + mutex)
   - `PostgresRepository` (con database/sql)
3. **Use Case**: Validaciones (hostname único, IP válida, status solo "running"/"stopped"/"maintenance")
4. **Handler HTTP**: API REST con los endpoints:
   - `GET /api/servers` — Listar todos
   - `GET /api/servers/{id}` — Obtener uno
   - `POST /api/servers` — Crear
   - `PATCH /api/servers/{id}` — Actualizar parcial
   - `DELETE /api/servers/{id}` — Eliminar
5. **Tests**: Mínimo 10 test cases (table-driven)
6. **Script Python**: Que llame a la API y genere un reporte JSON de todos los servidores
7. **PostgreSQL**: Ejecutar con Docker, crear las tablas con migraciones SQL

**Verificación**:

```bash
# Compilar sin errores
go build -o /tmp/server-api ./cmd/main.go

# Tests pasan
go test -v -cover ./...
# Cobertura mínima: 70%

# Race condition detector
go test -race ./...
# No debe reportar races

# La API responde
curl -s http://localhost:8080/api/servers | jq .
curl -s -X POST http://localhost:8080/api/servers \
  -H "Content-Type: application/json" \
  -d '{"hostname":"web-01","ip":"10.0.1.10","os":"Ubuntu 22.04","cpu":4,"ram":8}' | jq .

# PostgreSQL tiene datos
docker exec -it pg-lab psql -U rolando -d inventario -c "SELECT * FROM servers;"

# Script Python funciona
python3 report.py > /tmp/report.json && cat /tmp/report.json | jq .
```

---

## Recursos para esta Fase

### Go

1. **"The Go Programming Language" de Donovan & Kernighan** — EL libro de Go. Escrito por un co-creador de C. Claro, profundo.
2. **"Let's Go" de Alex Edwards** — Práctico, construís una app web real.
3. **Go Tour** (go.dev/tour) — Tutorial oficial interactivo (gratis)
4. **Go by Example** (gobyexample.com) — Referencia rápida con ejemplos (gratis)
5. **Effective Go** (go.dev/doc/effective_go) — Estilo y mejores prácticas oficiales (gratis)

### Python

1. **"Automate the Boring Stuff with Python" de Al Sweigart** — Gratis en automatetheboringstuff.com. Perfecto para automatización.
2. **Real Python** (realpython.com) — Tutoriales de calidad

### Bases de Datos

1. **"Designing Data-Intensive Applications" de Martin Kleppmann** (DDIA) — LA biblia. Denso pero fundamental. Leelo a lo largo de toda la guía.
2. **PostgreSQL Tutorial** (postgresqltutorial.com) — Gratis, paso a paso
3. **"The Art of PostgreSQL" de Dimitri Fontaine** — Avanzado, excelente

### Labs

- **Exercism** (exercism.org) — Ejercicios de Go con mentoring gratis
- **Go Playground** (go.dev/play) — Ejecutar Go en el navegador
- **SQLBolt** (sqlbolt.com) — SQL interactivo en el navegador

---

## Checkpoint: ¿Estoy listo para la Fase 4?

- [ ] ¿Puedo explicar qué es una interfaz y por qué es útil?
- [ ] ¿Puedo crear un programa con goroutines y channels?
- [ ] ¿Sé cuándo usar mutex vs channel?
- [ ] ¿Puedo escribir table-driven tests en Go?
- [ ] ¿Puedo crear una API REST en Go sin copiar código?
- [ ] ¿Puedo hacer CRUD con PostgreSQL?
- [ ] ¿Puedo explicar qué es un índice y para qué sirve?
- [ ] ¿Puedo escribir un script de Python que interactúe con una API?
- [ ] ¿Sé qué es una migración de base de datos?
- [ ] ¿Puedo detectar race conditions con `go test -race`?

Si respondiste 8+ de 10: avanzá a la Fase 4.
