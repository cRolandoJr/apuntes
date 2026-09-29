[[0. Índice DevOps]]

# Docker — Contenedores desde cero

> La base que el resto del vault da por sabida: [[Kubernetes]] orquesta contenedores,
> [[CI-CD con GitHub Actions]] los construye, [[Prometheus y Grafana]] corre en ellos.
> Todo eso asume que sabés esto. Es la skill #1 que pide un junior DevOps.

## 1. ¿Qué es un contenedor? (y qué NO es)

Un contenedor es un **proceso aislado** que empaqueta tu app + sus dependencias, usando
features del kernel Linux (namespaces = aislamiento de vista; cgroups = límites de
recursos). NO es una VM: no tiene kernel propio ni hardware virtual — comparte el kernel
del host. Por eso arranca en milisegundos y pesa MB, no GB.

```
VM                          Contenedor
┌──────────────┐            ┌──────────────┐
│ App          │            │ App          │
│ Libs         │            │ Libs         │
│ SO invitado  │  (GBs)     ├──────────────┤  (MBs)
│ Kernel guest │            │ (usa kernel  │
├──────────────┤            │  del host)   │
│ Hypervisor   │            ├──────────────┤
│ Kernel host  │            │ Kernel host  │
└──────────────┘            └──────────────┘
```

**Imagen vs contenedor:** la **imagen** es la plantilla inmutable (como una clase); el
**contenedor** es una instancia en ejecución (como un objeto). De una imagen levantás N
contenedores.

## 2. Docker vs Podman (tu caso)

`docker` y `podman` comparten CLI casi idéntica (`alias docker=podman` funciona para el
90%). Podman es **rootless** y **daemonless** por diseño (más seguro, sin proceso root
siempre corriendo). En tu setup NixOS usás **Podman con dockerCompat** — los comandos
`docker` de abajo funcionan igual. La diferencia práctica: en Podman no hace falta `sudo`.

## 3. El ciclo de vida (comandos que usás el 90% del tiempo)

```bash
docker pull nginx:alpine          # bajar una imagen del registry
docker images                     # listar imágenes locales
docker run -d --name web -p 8080:80 nginx:alpine
  # -d = detached (background) · --name = nombre · -p host:contenedor = mapeo de puerto
docker ps                         # contenedores corriendo (-a = incluye parados)
docker logs -f web                # ver logs (-f = follow, como tail -f)
docker exec -it web sh            # entrar a una shell DENTRO del contenedor
docker stop web && docker rm web  # parar y borrar
docker rmi nginx:alpine           # borrar la imagen
docker system prune -a            # limpiar todo lo no usado (libera disco)
```

**Gotcha del `-p`:** `-p 8080:80` = puerto 8080 del host → 80 del contenedor. El orden
importa: `host:contenedor`. Si no mapeás, el puerto del contenedor no es accesible desde afuera.

## 4. Dockerfile — construir tu propia imagen

Un Dockerfile es la receta. Cada instrucción crea una **capa** cacheada.

```dockerfile
FROM node:20-alpine              # imagen base (alpine = mínima)
WORKDIR /app                     # dir de trabajo dentro de la imagen
COPY package*.json ./            # copiar SOLO deps primero (ver caching abajo)
RUN npm ci                       # instalar dependencias
COPY . .                         # ahora sí, el resto del código
EXPOSE 3000                      # puerto informativo (documenta, no publica)
CMD ["node", "server.js"]        # comando por defecto al arrancar
```

```bash
docker build -t mi-app:1.0 .     # construir (-t = tag; . = contexto = dir actual)
docker run -p 3000:3000 mi-app:1.0
```

### Caching de capas — la optimización que más importa
Docker cachea cada capa; si una capa no cambió, reusa la caché. Por eso se copian las
**dependencias antes que el código**: si solo cambiás una línea de código pero no las
deps, `npm ci` no se re-ejecuta (capa cacheada). Copiar todo junto (`COPY . .` primero)
invalida la caché en cada cambio → builds lentísimos.

### `.dockerignore` (obligatorio)
Como `.gitignore`, evita mandar basura al build context (que se sube entero al daemon):
```
.git
node_modules
*.log
.env
dist
```

## 5. Multi-stage builds — imágenes chicas y seguras

El patrón profesional: compilar en una imagen con toolchain, copiar solo el artefacto a
una imagen final mínima. Diferencia real: **~350MB → ~15MB**, y sin compilador dentro
(menos superficie de ataque).

```dockerfile
# STAGE 1: build
FROM golang:1.25-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/bin ./cmd/app   # binario estático

# STAGE 2: runtime mínimo
FROM alpine:3.20                 # o gcr.io/distroless/static (aún menos)
COPY --from=build /app/bin /bin  # SOLO el binario cruza de stage
EXPOSE 8080
CMD ["/bin"]
```

## 6. Volúmenes y redes

**Volúmenes** — persistir datos (los contenedores son efímeros; sin volumen, al borrar
el contenedor perdés los datos):
```bash
docker volume create pgdata
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16
  # named volume (gestionado por docker) ↑
docker run -v $(pwd)/config:/etc/app:ro mi-app   # bind mount (carpeta del host, :ro = read-only)
```

**Redes** — los contenedores en la misma red se ven por nombre (DNS interno):
```bash
docker network create appnet
docker run -d --name db --network appnet postgres:16
docker run -d --name api --network appnet mi-api   # la api llega a la db por "db:5432"
```

## 7. Docker Compose — múltiples contenedores declarativos

Para una app con varios servicios (api + db + cache), un `compose.yaml` reemplaza N
comandos `docker run`. Es el paso previo mental a Kubernetes.

```yaml
# compose.yaml  (Compose v2: se invoca "docker compose", sin guion)
services:
  api:
    build: .                     # construye desde el Dockerfile local
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db                # llega a la db por el nombre del servicio
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d             # levantar todo (v2: "compose", NO "docker-compose")
docker compose ps                # estado
docker compose logs -f api       # logs de un servicio
docker compose down              # parar y borrar (down -v = también los volúmenes)
```

> **Gotcha de versión:** `docker-compose` (con guion) es la v1 en Python, deprecada.
> Lo actual es `docker compose` (subcomando, v2 en Go). Si un tutorial usa el guion, es viejo.

## 8. Buenas prácticas (lo que preguntan en entrevista)

- **Multi-stage** siempre que compiles (Go, Node build, Rust).
- **Imagen base mínima**: `alpine` o `distroless`, no `ubuntu` completo.
- **Un proceso por contenedor** (no meter nginx + app + db en uno).
- **No correr como root**: `USER appuser` en el Dockerfile.
- **No hornear secretos** en la imagen (van por env/secrets en runtime, ver [[Secrets Management]]).
- **Tags específicos** (`node:20-alpine`, no `node:latest` — `latest` no es reproducible).
- **`.dockerignore`** siempre, y **commitear** el Dockerfile.
- **Escaneo de imágenes** (Trivy/Grype) antes de push — supply chain.

## 9. Troubleshooting rápido

| Síntoma | Causa probable |
|---|---|
| `port is already allocated` | Otro contenedor/proceso usa ese puerto del host. `docker ps` + cambiar `-p`. |
| Contenedor sale al instante (`Exited (0)`) | El CMD terminó — un contenedor vive mientras su proceso principal viva. |
| Cambios de código no aparecen | Rebuildeaste sin `--no-cache` o el caching reusó una capa vieja. |
| Imagen enorme | Falta multi-stage / base pesada / copiaste `node_modules` (falta `.dockerignore`). |
| `permission denied` en un volumen | UID del proceso del contenedor ≠ dueño del archivo en el host (común con bind mounts). |

## Relacionados
- [[Kubernetes]] — orquesta estos contenedores a escala.
- [[CI-CD con GitHub Actions]] — construye y pushea imágenes en el pipeline.
- [[../Sysadmin/Docker y docker compose|Docker (nota Sysadmin)]] — el Dockerfile Go multi-stage con más detalle.
