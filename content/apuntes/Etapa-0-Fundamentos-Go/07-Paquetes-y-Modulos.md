# Paquetes y Módulos

## go.mod — el archivo raíz del módulo

```go
module github.com/tuusuario/gestion_productos

go 1.22

require (
    github.com/99designs/gqlgen v0.17.45
    github.com/google/uuid v1.6.0
    github.com/jackc/pgx/v5 v5.5.4
)

require (
    // dependencias indirectas (de tus dependencias)
    github.com/vektah/gqlparser/v2 v2.5.11 // indirect
)
```

---

## Comandos esenciales

```bash
# Inicializar un módulo nuevo
go mod init github.com/usuario/proyecto

# Agregar una dependencia
go get github.com/gin-gonic/gin@latest
go get github.com/jackc/pgx/v5@v5.5.4  # versión específica

# Limpiar dependencias no usadas / agregar faltantes
go mod tidy

# Descargar dependencias sin compilar
go mod download

# Ver árbol de dependencias
go mod graph

# Verificar integridad de go.sum
go mod verify

# Ver qué versión de una dependencia se está usando y por qué
go mod why github.com/jackc/pgx/v5
```

---

## Estructura de paquetes — convenciones

```
gestion_productos/
├── cmd/
│   └── main.go              ← Entry point. Construye las dependencias (DI manual)
├── internal/                ← Solo importable desde DENTRO de este módulo
│   ├── domain/
│   │   ├── product.go       ← Entidades + interfaces del dominio
│   │   └── errors.go        ← Tipos de error de dominio
│   ├── usecase/
│   │   └── product_usecase.go
│   ├── repository/
│   │   ├── memory_repository.go
│   │   └── postgres_repository.go
│   └── delivery/
│       ├── graphql/
│       │   └── resolver.go
│       └── http/
│           └── handler.go
├── pkg/                     ← Código público reutilizable (librería interna)
│   └── middleware/
│       └── logging.go
├── graph/
│   └── schema.graphqls
├── migrations/              ← Archivos SQL de migraciones
│   ├── 001_create_products.up.sql
│   └── 001_create_products.down.sql
├── go.mod
├── go.sum
└── Makefile
```

---

## internal/ — protección de paquetes

El directorio `internal/` es especial: solo puede ser importado por código dentro del mismo módulo.

```go
// Esto compila desde dentro del módulo:
import "github.com/tuusuario/gestion_productos/internal/domain"

// Esto NO compila desde un módulo externo:
// import "github.com/tuusuario/gestion_productos/internal/domain"
// error: use of internal package not allowed
```

---

## Visibilidad — exportado vs no exportado

En Go la visibilidad se controla con mayúscula/minúscula al primer carácter.

```go
package user

// Exportado — accesible desde otros paquetes
type User struct {
    ID    string  // exportado
    Name  string  // exportado
    email string  // NO exportado (privado al paquete)
}

// Exportado
func NewUser(id, name, email string) *User {
    return &User{ID: id, Name: name, email: email}
}

// No exportado — función auxiliar privada al paquete
func validateEmail(email string) bool {
    return strings.Contains(email, "@")
}
```

---

## Importar paquetes

```go
import (
    // stdlib
    "fmt"
    "strings"
    "time"

    // forma de poner alias para evitar conflictos de nombres
    gqlgraphql "github.com/99designs/gqlgen/graphql"

    // importar solo por side effects (ejecuta init())
    _ "github.com/lib/pq"  // registra el driver de PostgreSQL
)
```

### El blank identifier `_` en imports

```go
// Drivers de base de datos se registran con init() al importar
import _ "github.com/lib/pq"           // PostgreSQL
import _ "github.com/mattn/go-sqlite3" // SQLite
```

---

## Naming conventions

```go
// Paquetes: singular, minúsculas, sin guiones ni underscores
package user      // bien
package users     // evitar plural
package user_repo // evitar underscore
package userrepo  // bien si hay que combinar

// No repetir el nombre del paquete en el nombre de la función/tipo
package user
type UserUser struct{ ... }  // redundante — se usa como user.UserUser
type User struct{ ... }      // correcto — se usa como user.User

func NewUserUser() *User {}  // redundante
func New() *User {}          // correcto — se usa como user.New()
```

---

## init() — inicialización de paquete

```go
// Se ejecuta automáticamente cuando el paquete es importado
// Antes de main()
// Múltiples init() en el mismo paquete se ejecutan en orden de aparición

var cache *Cache

func init() {
    cache = NewCache()
    // registrar drivers, cargar configuración, etc.
}
```

> Evitar `init()` para lógica compleja. Hace difícil testear. Preferible inicialización explícita en `main()`.

---

## go:generate — automatizar generación de código

```go
// En el archivo que dispara la generación:
//go:generate go run github.com/99designs/gqlgen generate
//go:generate mockgen -source=domain/product.go -destination=mocks/product_mock.go

// Correr:
// go generate ./...
```

---

## Makefile — automatizar tareas comunes

```makefile
.PHONY: build run test lint generate tidy

build:
	go build -o bin/server ./cmd/main.go

run:
	go run ./cmd/main.go

test:
	go test -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

lint:
	golangci-lint run ./...

generate:
	go generate ./...

tidy:
	go mod tidy

docker-build:
	docker build -t gestion-productos:latest .
```

---

## Práctica: Novato vs Profesional

### Novato — todo en un paquete

```
main.go   ← 800 líneas, todo mezclado: handlers, DB, lógica, config
```

### Profesional — separación por responsabilidad

```
Reglas:
1. cmd/ solo arranca el servidor y conecta dependencias
2. internal/domain solo conoce el negocio, nunca importa paquetes de infraestructura
3. internal/usecase importa solo domain
4. internal/repository importa domain + driver de DB
5. internal/delivery importa domain + usecase
6. main.go importa todos los anteriores para "cablearlos"

Imports permitidos:
domain      → solo stdlib
usecase     → domain + stdlib
repository  → domain + stdlib + driver DB
delivery    → domain + usecase + stdlib + framework web
cmd/main.go → todo lo anterior (es el punto de entrada)
```

```go
// cmd/main.go — el único lugar donde se conocen todas las capas
func main() {
    db := setupDB()

    repo := postgres.NewProductRepository(db)       // capa de datos
    uc := usecase.NewProductUseCase(repo)            // lógica de negocio
    resolver := graphql.NewResolver(uc)              // entrega

    srv := setupGraphQLServer(resolver)
    log.Fatal(http.ListenAndServe(":8080", srv))
}
```
