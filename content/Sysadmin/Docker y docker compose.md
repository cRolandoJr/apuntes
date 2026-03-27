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

```bash
# 1. Imagen Base (Debe coincidir con la versión de tu go.mod)
FROM golang:1.25-alpine

# 2. Directorio de trabajo dentro del contenedor
WORKDIR /app

# 3. Copiar dependencias primero (aprovecha la caché de Docker)
COPY go.mod ./
# COPY go.sum ./

# 4. Descargar librerías
RUN go mod download

# 5. Copiar el resto del código
COPY . .

# 6. Compilar (Apuntando a la carpeta donde está el main.go)
RUN go build -o gochat ./cmd/gochat

# 7. Puerto informativo
EXPOSE 8080

# 8. Comando de arranque
CMD ["./gochat"]
```

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
