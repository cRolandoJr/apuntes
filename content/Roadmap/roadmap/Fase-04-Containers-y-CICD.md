# Fase 4: Containers & CI/CD — Meses 10 a 12

> **Objetivo**: Dominar Docker en profundidad (no solo `docker run`), entender container runtimes, y construir pipelines de CI/CD completos. Esto es el puente entre "escribir código" y "entregar software".

---

## Mes 10: Docker en Profundidad

### 10.1 ¿Qué es un Container Realmente?

Un container NO es una VM liviana. Es un **proceso de Linux aislado** usando features del kernel:

```
VM:
┌──────────────┐ ┌──────────────┐
│   App A      │ │   App B      │
│   Libs       │ │   Libs       │
│   Guest OS   │ │   Guest OS   │  ← Kernel completo para cada VM
├──────────────┤ ├──────────────┤
│      Hypervisor (VMware, KVM) │
├───────────────────────────────┤
│         Host OS + Kernel      │
└───────────────────────────────┘

Container:
┌──────────┐ ┌──────────┐
│  App A   │ │  App B   │
│  Libs    │ │  Libs    │  ← Solo las libs necesarias
├──────────┴─┴──────────┤
│     Container Runtime  │
├───────────────────────┤
│   Host OS + Kernel     │  ← COMPARTEN el mismo kernel
└───────────────────────┘
```

**Tecnologías del kernel que hacen posible los containers**:

1. **Namespaces**: Aíslan lo que el proceso PUEDE VER
   - `pid` — Solo ve sus propios procesos
   - `net` — Tiene su propia interfaz de red, IP, puertos
   - `mnt` — Tiene su propio árbol de directorios
   - `uts` — Tiene su propio hostname
   - `user` — Tiene sus propios UIDs (root dentro ≠ root fuera)
   - `ipc` — Aislamiento de memoria compartida

2. **cgroups (Control Groups)**: Limitan lo que el proceso PUEDE USAR
   - CPU: "máximo 2 cores"
   - Memoria: "máximo 512MB"
   - I/O: "máximo 100MB/s de disco"
   - Red: "máximo 10Mbps"

3. **Union Filesystem (OverlayFS)**: Capas de solo lectura + una capa de escritura
   ```
   Capa de escritura (container) ← tus cambios van aquí
   ─────────────────────────────
   Capa 3: COPY . /app          ← tu código
   Capa 2: RUN go build         ← compilación
   Capa 1: FROM golang:1.25     ← imagen base
   ```
   Si 10 containers usan la misma imagen base, comparten las capas inferiores.

**Ejercicio verificable — Ver los namespaces de un container**:

```bash
# Ejecutar un container
docker run -d --name demo alpine sleep 3600

# Ver el PID real del proceso en el host
docker inspect --format '{{.State.Pid}}' demo

# Ver sus namespaces
ls -la /proc/$(docker inspect --format '{{.State.Pid}}' demo)/ns/
# Verás: cgroup, ipc, mnt, net, pid, user, uts

# Desde dentro del container:
docker exec demo ps aux
# Solo ve UN proceso (sleep). Desde el host, es un proceso más.

# Ver cgroups
cat /sys/fs/cgroup/docker/$(docker inspect --format '{{.Id}}' demo)/memory.max 2>/dev/null || \
cat /proc/$(docker inspect --format '{{.State.Pid}}' demo)/cgroup

docker rm -f demo
```

### 10.2 Dockerfile — Mejores Prácticas

Tu Dockerfile actual funciona, pero podemos mejorarlo mucho:

```dockerfile
# === MALO (lo que suele hacer un junior) ===
FROM golang:1.25.5
WORKDIR /app
COPY . .
RUN go build -o main ./cmd/main.go
CMD ["./main"]
# Problemas: imagen de 800MB+, root, cache ineficiente

# === BUENO (producción) ===
# Stage 1: Build
FROM golang:1.25.5-alpine AS builder

WORKDIR /build

# Copiar PRIMERO go.mod y go.sum → cachear dependencias
COPY go.mod go.sum ./
RUN go mod download
# Si go.mod no cambió, Docker reutiliza esta capa cacheada
# Aunque cambies el código, no re-descarga dependencias

# Copiar el resto del código
COPY . .

# Compilar binario estático
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app ./cmd/main.go
# -ldflags="-s -w" → strip debug info → binario más chico

# Stage 2: Runtime
FROM alpine:3.19

# Instalar certificados CA (para HTTPS)
RUN apk --no-cache add ca-certificates

# Usuario no-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --from=builder /app .

# Cambiar a usuario no-root
USER appuser

EXPOSE 8081

# Healthcheck: Docker verifica que el servicio funciona
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8081/ || exit 1

ENTRYPOINT ["./app"]
```

**Key insights**:

- **Orden de COPY**: Primero `go.mod/go.sum`, después el código. Así Docker cachea las dependencias.
- **Multi-stage**: Build en imagen grande, ejecutar en imagen mínima.
- **Non-root user**: NUNCA correr como root en producción.
- **HEALTHCHECK**: Docker puede reiniciar containers unhealthy.
- **`-ldflags="-s -w"`**: Reduce el binario un ~30%.

### 10.3 Docker Compose Avanzado

```yaml
# docker-compose.yml — Stack completo
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=app
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - DB_NAME=productos
    depends_on:
      postgres:
        condition: service_healthy # Esperar a que PG esté listo
    restart: unless-stopped
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 256M

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: productos
    volumes:
      - pgdata:/var/lib/postgresql/data # Datos persistentes
      - ./migrations:/docker-entrypoint-initdb.d # SQL al primer arranque
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d productos"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend

volumes:
  pgdata: # Volumen nombrado: persiste datos entre restarts

networks:
  backend:
    driver: bridge # Red interna: api puede hablar con postgres por nombre

secrets:
  db_password:
    file: ./secrets/db_password.txt # Nunca hardcodes passwords
```

```bash
# Comandos esenciales
docker compose up -d              # Levantar todo en background
docker compose logs -f api        # Seguir logs del servicio api
docker compose ps                 # Estado de los servicios
docker compose exec api sh        # Shell dentro del container api
docker compose down               # Parar todo
docker compose down -v            # Parar todo Y borrar volúmenes (datos!)
docker compose build --no-cache   # Rebuild sin cache
```

### 10.4 Docker Networking

```bash
# Los containers en la misma red se comunican por NOMBRE
# En tu compose, "api" habla con "postgres" usando el hostname "postgres"
# No necesitan IPs hardcodeadas

# Ver redes
docker network ls
docker network inspect bridge

# Tipos de red:
# bridge  → Red interna (default). Containers se ven entre sí.
# host    → Container usa la red del host directamente. Sin aislamiento.
# none    → Sin red. Para containers que no necesitan conectividad.
# overlay → Multi-host (para Docker Swarm o Kubernetes)

# Debug de red dentro de un container
docker run --rm -it --network=tu_red nicolaka/netshoot
# netshoot tiene todas las herramientas de red (dig, nmap, tcpdump, curl...)
```

### 10.5 Seguridad en Containers

```bash
# 1. NUNCA correr como root
# En Dockerfile: USER appuser
# Verificar:
docker exec tu_container whoami  # Debería decir "appuser", NO "root"

# 2. Escanear vulnerabilidades
docker scout quickview tu_imagen      # Docker Scout (built-in)
# o
trivy image tu_imagen                 # Trivy (instalable)

# 3. Usar imágenes base mínimas
# De mejor a peor:
# scratch        → Nada. Solo tu binario estático. Ideal para Go.
# distroless     → Google. Mínimo runtime.
# alpine         → 5MB. Tiene package manager.
# slim           → Debian recortado. ~80MB
# Ubuntu/Debian  → Completo. 100MB+. Solo para desarrollo.

# 4. No almacenar secretos en la imagen
# NUNCA: ENV DB_PASSWORD=secreto123
# SÍ: Usar Docker secrets, variables de entorno en runtime, o vault

# 5. Imagen inmutable: NUNCA instalar nada en runtime
# Todo lo que necesita el container debe estar en la imagen
```

**Ejercicio verificable**:

```bash
# Optimizar tu imagen actual
cd /home/Rolando/gestion_productos

# Tamaño ANTES
docker build -t gestion:before .
docker images gestion:before --format "{{.Size}}"

# Construir con la versión optimizada (el Dockerfile bueno de arriba)
# (editá tu Dockerfile)
docker build -t gestion:after .
docker images gestion:after --format "{{.Size}}"
# El after debería ser MUCHO más chico

# Verificar que no corre como root
docker run -d --name test gestion:after
docker exec test whoami
# Debería decir "appuser"
docker rm -f test

# Escanear vulnerabilidades
docker scout quickview gestion:after
```

---

## Mes 11: CI/CD (Continuous Integration / Continuous Delivery)

### 11.1 ¿Qué es CI/CD?

```
Desarrollador ─push→ GitHub ─trigger→ Pipeline CI/CD
                                        │
                                        ├─ 1. Build (compilar)
                                        ├─ 2. Test (ejecutar tests)
                                        ├─ 3. Lint (calidad de código)
                                        ├─ 4. Security Scan (vulnerabilidades)
                                        ├─ 5. Build Image (Docker)
                                        ├─ 6. Push Image (registry)
                                        └─ 7. Deploy (producción)
```

**CI (Continuous Integration)**: Cada push se compila, testea y valida automáticamente.
**CD (Continuous Delivery)**: El código validado se puede deployar con un click.
**CD (Continuous Deployment)**: El código validado se deploya AUTOMÁTICAMENTE.

### 11.2 GitHub Actions — Tu Primera Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25"

      - name: Descargar dependencias
        run: go mod download

      - name: Ejecutar tests
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Verificar cobertura
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total: | awk '{print $3}' | tr -d '%')
          echo "Cobertura: ${COVERAGE}%"
          # Fallar si cobertura < 60%
          if (( $(echo "$COVERAGE < 60" | bc -l) )); then
            echo "Cobertura insuficiente: ${COVERAGE}% < 60%"
            exit 1
          fi

      - name: Lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest

  build:
    needs: test # Solo si test pasó
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t gestion-productos:${{ github.sha }} .

      - name: Escanear vulnerabilidades
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "gestion-productos:${{ github.sha }}"
          severity: "CRITICAL,HIGH"
          exit-code: "1" # Fallar si hay vulnerabilidades críticas

  deploy:
    needs: build # Solo si build pasó
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' # Solo en main, no en PRs
    steps:
      - name: Deploy a producción
        run: echo "Aquí iría el deploy real (SSH, K8s, etc.)"
```

### 11.3 Estructura de un Pipeline Profesional

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│    BUILD     │──→│    TEST     │──→│   SECURITY  │──→│   DEPLOY    │
│             │   │             │   │             │   │             │
│ go build    │   │ go test     │   │ trivy scan  │   │ staging     │
│ docker build│   │ go vet      │   │ lint        │   │ prod        │
│             │   │ coverage    │   │ SAST/DAST   │   │             │
└─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘
      │                 │                 │                 │
      │   Si falla      │  Si falla       │  Si falla       │
      └── STOP ─────────┴── STOP ─────────┴── STOP ─────────┘
```

### 11.4 Registries (Dónde Guardar tus Imágenes)

```bash
# Docker Hub (público/privado)
docker login
docker tag gestion-productos:latest tuusuario/gestion-productos:v1.0.0
docker push tuusuario/gestion-productos:v1.0.0

# GitHub Container Registry (ghcr.io)
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker tag gestion-productos:latest ghcr.io/tuusuario/gestion-productos:v1.0.0
docker push ghcr.io/tuusuario/gestion-productos:v1.0.0

# Versionado semántico de imágenes:
# v1.0.0      → Release estable
# v1.0.0-rc1  → Release candidate
# latest      → Última versión (NO usar en producción — es ambiguo)
# sha-abc1234 → Versión por commit (exacta, reproducible)
```

---

## Mes 12: Docker Avanzado y Proyecto Final

### 12.1 Multi-Architecture Builds

```bash
# Construir para múltiples arquitecturas (ARM para Raspberry Pi, etc.)
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t tuusuario/gestion-productos:v1.0.0 --push .
```

### 12.2 Docker en Producción — Lo Que Nadie Te Dice

1. **Los logs van a stdout/stderr** — Docker los captura automáticamente
2. **Un proceso por container** — No corras nginx + tu app en el mismo container
3. **Los containers son efímeros** — Pueden morir y recrearse en cualquier momento
4. **Los datos van en volúmenes** — Nunca dentro del container filesystem
5. **Las imágenes son inmutables** — La misma imagen en dev, staging y producción
6. **Limitar recursos** — Siempre poné límites de CPU y memoria
7. **Health checks** — Docker necesita saber si tu app está sana

---

## Proyecto Integrador de Fase 4

### "Pipeline CI/CD Completo"

1. **Repositorio en GitHub** con tu API de productos
2. **Dockerfile** optimizado (multi-stage, non-root, healthcheck)
3. **Docker Compose** con API + PostgreSQL + volúmenes + healthchecks
4. **GitHub Actions pipeline** con:
   - Tests con cobertura mínima del 60%
   - Lint con golangci-lint
   - Build de imagen Docker
   - Scan de vulnerabilidades con Trivy
   - Push a GitHub Container Registry
5. **Makefile** para automatizar comandos locales:

   ```makefile
   .PHONY: build test lint docker-build

   build:
       go build -o bin/api ./cmd/main.go

   test:
       go test -v -race -coverprofile=coverage.out ./...

   lint:
       golangci-lint run

   docker-build:
       docker build -t gestion-productos:dev .

   up:
       docker compose up -d

   down:
       docker compose down
   ```

**Verificación**:

```bash
# Pipeline pasa en GitHub
# Ir a GitHub → Actions → verificar que el workflow terminó en verde

# Localmente:
make test                    # Tests pasan
make lint                    # Sin warnings
make docker-build            # Build exitoso
make up                      # Stack levanta
curl http://localhost:8081/  # API responde
docker compose ps            # Todos los servicios "healthy"

# Imagen en registry
docker pull ghcr.io/tuusuario/gestion-productos:latest
```

---

## Recursos para esta Fase

### Docker

1. **Documentación oficial de Docker** (docs.docker.com) — La mejor fuente, siempre actualizada
2. **"Docker Deep Dive" de Nigel Poulton** — Profundo, va más allá de lo básico
3. **"Container Security" de Liz Rice** — Seguridad de containers. Excelente.
4. **Docker labs** (github.com/docker/labs) — Ejercicios oficiales

### CI/CD

1. **GitHub Actions Docs** (docs.github.com/en/actions) — Referencia completa
2. **"Continuous Delivery" de Jez Humble & David Farley** — El libro clásico de CD

### Labs

- **KillerCoda** (killercoda.com) — Labs de Docker interactivos
- **Play with Docker** (labs.play-with-docker.com) — Docker en el navegador, gratis

---

## Checkpoint: ¿Estoy listo para la Fase 5?

- [ ] ¿Puedo explicar qué son namespaces y cgroups?
- [ ] ¿Puedo escribir un Dockerfile multi-stage optimizado?
- [ ] ¿Puedo crear un docker-compose con múltiples servicios?
- [ ] ¿Puedo explicar por qué no correr como root en containers?
- [ ] ¿Puedo crear una pipeline de CI/CD con GitHub Actions?
- [ ] ¿Puedo pushear imágenes a un registry?
- [ ] ¿Puedo usar volúmenes y redes en Docker Compose?
- [ ] ¿Puedo escanear una imagen por vulnerabilidades?
- [ ] ¿Puedo debuggear un container que no arranca?
- [ ] ¿Puedo explicar la diferencia entre un container y una VM?

Si respondiste 8+ de 10: avanzá a la Fase 5.
