[[0. General Tips]]

### 1. Conceptos Fundamentales

Una analogía de construcción para entender la arquitectura:

- **Dockerfile (La Receta):** Archivo de texto que describe cómo crear la imagen. (Ej: "Usa Linux Alpine, instala Go, copia estos archivos").
- **Imagen (El Molde):** El resultado de la receta. Es inmutable.
- **Contenedor (El Ladrillo):** Una instancia viva de la imagen.
- **Docker Compose (El Arquitecto):** Archivo YAML que orquesta todo. Define puertos, volúmenes y reinicios automáticos.

### 2. Estructura de Archivos (Layout Go)

Para que Docker vea tanto el `go.mod` como el código en subcarpetas, el `Dockerfile` debe estar en la **RAÍZ** del proyecto.

```
/mi-proyecto
├── Dockerfile        <-- AQUI (en la raíz)
├── compose.yaml      <-- AQUI (en la raíz)
├── go.mod
└── cmd/
    └── gochat/
        └── main.go
```

### 3. Dockerfile (Ejemplo Go)

Un Dockerfile Go debe ser **multi-stage**: se compila en una imagen con el toolchain
(cientos de MB) pero la imagen final solo lleva el binario (MB). Es lo primero que
preguntan sobre Docker.

```dockerfile
# ── STAGE 1: build (tiene el compilador de Go) ────────────────────────
FROM golang:1.25-alpine AS build
WORKDIR /app

# Copiar dependencias primero: si no cambian, Docker cachea esta capa
COPY go.mod go.sum ./          # go.sum SÍ (builds reproducibles), no comentado
RUN go mod download

COPY . .
# CGO_ENABLED=0 → binario estático (corre en scratch/distroless sin libc)
RUN CGO_ENABLED=0 go build -o /gochat ./cmd/gochat

# ── STAGE 2: runtime (imagen final mínima, sin toolchain) ─────────────
FROM alpine:3.20           # o gcr.io/distroless/static para aún menos superficie
WORKDIR /app
COPY --from=build /gochat ./gochat    # solo el binario cruza de stage
EXPOSE 8080
# HEALTHCHECK opcional para que Docker sepa si la app está viva
CMD ["./gochat"]
```

> **Por qué importa:** single-stage = imagen de ~350MB con el toolchain de Go dentro
> (y toda su superficie de ataque). Multi-stage = ~15MB con solo tu binario. Necesitás
> un `.dockerignore` (excluir `.git`, `node_modules`, etc.) para no mandar basura al build.

### 4. Docker Compose (`compose.yaml`)

La forma profesional de ejecutar contenedores.

```yaml
services:
  app-chat:
    # Construir usando el Dockerfile de la carpeta actual (.)
    build: .
    container_name: produccion-chat
    # Si se cae o reinicio el servidor, levántate solo
    restart: always
    # Mapeo: Puerto_Real_Debian : Puerto_Interno_Contenedor
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
    logging:
      driver: "journald"
      options:
        tag: "{{.Name}}"
```

> **Logging:** La sección `logging` envía la salida de la app directamente al journal de Linux, unificando los logs del contenedor con los del sistema.

### 5. Comandos de Supervivencia

| **Acción**                                        | **Comando**                                    |
| ------------------------------------------------- | ---------------------------------------------- |
| **Levantar todo** (y reconstruir si hubo cambios) | `sudo docker compose up --build -d`            |
| **Ver estado**                                    | `sudo docker compose ps`                       |
| **Ver logs** (Salida de la app)                   | `sudo docker compose logs -f`                  |
| **Apagar todo**                                   | `sudo docker compose down`                     |
| **Entrar al contenedor** (Shell interactiva)      | `sudo docker exec -it produccion-chat /bin/sh` |

## Observabilidad (logs)

Como configuramos el driver `journald`, tenemos dos formas de ver qué pasa:

1. **Vía Linux (Recomendado/Profesional):** Ver logs unificados con el sistema.

```bash
sudo journalctl -t (nombre_del_contenedor) -f
```

2. **Vía Docker (Clásico):** Si no usas journald, este es el estándar.

```bash
docker compose logs -f
```

## Resolución de Conflictos Comunes

**Error:** `bind: address already in use`

- **Causa:** Intentas levantar Docker en el puerto 8080, pero ya hay otro programa (ej: tu app corriendo manualmente) usándolo.
- **Solución:**
  1. Encontrar al culpable: `sudo ss -lptn 'sport = :8080'`
  2. Eliminarlo: `sudo kill -9 <PID>` o `sudo systemctl stop <servicio>`
  3. Reintentar Docker.
