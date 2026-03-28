# Docker para proyectos Go

Docker empaqueta tu aplicación + sus dependencias en una imagen reproducible. En Go, el resultado final es un binario estático — ideal para imágenes mínimas.

---

## Multi-stage Dockerfile — el estándar profesional

```dockerfile
# ================================================================
# STAGE 1: Builder — imagen completa con toolchain de Go
# ================================================================
FROM golang:1.22-alpine AS builder

# Instalar dependencias necesarias para compilar (ca-certificates para HTTPS)
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /app

# Copiar módulos primero — aprovecha cache de Docker
# Si go.mod/go.sum no cambiaron, esta capa se reutiliza
COPY go.mod go.sum ./
RUN go mod download

# Copiar el código fuente
COPY . .

# Compilar — flags importantes:
# CGO_ENABLED=0 → binario estático (no depende de libc)
# -ldflags="-s -w" → elimina debug symbols (binario más pequeño)
# -trimpath → elimina rutas absolutas del binario (seguridad)
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-s -w -X main.version=${VERSION:-dev}" \
    -trimpath \
    -o /app/server \
    ./cmd/main.go

# ================================================================
# STAGE 2: Runner — imagen mínima para producción
# ================================================================
FROM scratch AS runner

# Copiar solo lo necesario del builder
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /app/server /server

# Copiar migraciones si las ejecutás al iniciar
COPY --from=builder /app/migrations /migrations

# Usuario no-root — buena práctica de seguridad
USER 65534

EXPOSE 8080

# Health check
HEALTHCHECK --interval=15s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/server", "-healthcheck"] || exit 1

ENTRYPOINT ["/server"]
```

### Tamaños comparados

| Base Image           | Tamaño binario Go                   |
| -------------------- | ----------------------------------- |
| `ubuntu:22.04`       | ~180 MB                             |
| `golang:1.22-alpine` | ~350 MB (solo builder)              |
| `alpine:3.20`        | ~20 MB                              |
| `scratch`            | ~10 MB (solo el binario + certs)    |
| `distroless/static`  | ~15 MB (más compatible que scratch) |

---

## .dockerignore — no copiar lo que no hace falta

```
# .dockerignore
.git
.github
*.md
.env
.env.*
docker-compose*.yml
docs/
tmp/
vendor/

# Test files
*_test.go
testdata/
```

---

## docker-compose — entorno local completo

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder # usar el stage builder en desarrollo (tiene hot-reload)
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=gestion_productos
      - DB_USER=app_user
      - DB_PASSWORD=app_password
      - DB_SSLMODE=disable
      - JWT_SECRET=dev-secret-change-in-prod
      - LOG_LEVEL=debug
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - .:/app # hot-reload en desarrollo

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: gestion_productos
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d # ejecutar al crear el contenedor
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d gestion_productos"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  # Opcional: herramientas de desarrollo
  adminer:
    image: adminer
    ports:
      - "8090:8080"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

```bash
# Comandos útiles
docker compose up -d              # levantar en background
docker compose up --build         # rebuild y levantar
docker compose logs -f app        # ver logs del servicio
docker compose exec postgres psql -U app_user -d gestion_productos
docker compose down               # bajar
docker compose down -v            # bajar + borrar volumes (reset DB)
```

---

## Hot-reload en desarrollo — Air

```bash
go install github.com/air-verse/air@latest
```

```toml
# .air.toml
root = "."
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/server ./cmd/main.go"
  bin = "./tmp/server"
  include_ext = ["go", "tpl", "tmpl", "html", "graphqls"]
  exclude_dir = ["tmp", "vendor"]
  delay = 1000  # ms
```

```yaml
# En docker-compose.yml, para el servicio app en dev:
app:
  image: cosmtrek/air
  working_dir: /app
  volumes:
    - .:/app
```

---

## Dockerfile para producción con secrets

```dockerfile
# Nunca pasar secrets como ARG (quedan en las capas del historial)
# MAL:
ARG DB_PASSWORD=secret123

# BIEN: usar Docker secrets o variables de entorno en runtime
# Los secrets se montan como archivos, no quedan en la imagen
RUN --mount=type=secret,id=db_password \
    DB_PASSWORD=$(cat /run/secrets/db_password) && ...
```

---

## Práctica: Novato vs Profesional

### Novato

```dockerfile
# Una sola stage — imagen de 350MB
FROM golang:1.22
COPY . .
RUN go build -o server .
CMD ["./server"]

# Problemás:
# - Imagen enorme (herramientas de desarrollo incluidas)
# - Código fuente y herramientas de build en producción
# - Corriendo como root
# - Sin health check
```

### Profesional

```dockerfile
# Multi-stage: 350MB builder → 10MB runner
# CGO_ENABLED=0 → binario estático
# USER 65534 → no-root
# HEALTHCHECK → Kubernetes sabe si está listo
# .dockerignore → build context limpio
# Capas ordenadas de más estable a más cambiante → cache óptimo
```
