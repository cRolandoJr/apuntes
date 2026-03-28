# CI/CD con GitHub Actions para Go

Un pipeline de CI/CD automatiza: validar el código → testear → construir → publicar. Cada push debe pasar por este proceso antes de llegar a producción.

---

## Pipeline completo — go.yml

```yaml
# .github/workflows/go.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ================================================================
  # JOB 1: Lint — verificar calidad de código
  # ================================================================
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true # cachea GOMODCACHE automáticamente

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

  # ================================================================
  # JOB 2: Test — correr todos los tests con cobertura
  # ================================================================
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: test_db
          DB_USER: test_user
          DB_PASSWORD: test_pass
          DB_SSLMODE: disable
        run: |
          go test -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Cobertura insuficiente: $COVERAGE% (mínimo 70%)"
            exit 1
          fi
          echo "Cobertura: $COVERAGE%"

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.out

  # ================================================================
  # JOB 3: Build — compilar el binario
  # ================================================================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Build
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
          go build \
            -ldflags="-s -w -X main.version=${{ github.sha }}" \
            -trimpath \
            -o server \
            ./cmd/main.go

      - name: Upload binary artifact
        uses: actions/upload-artifact@v4
        with:
          name: server-binary
          path: server
          retention-days: 1

  # ================================================================
  # JOB 4: Docker — construir y publicar imagen
  # Solo en push a main
  # ================================================================
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: [build]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write # para publicar en GHCR

    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }} # automático, no necesitás configurarlo

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha # cache de GitHub Actions
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.sha }}
```

---

## golangci-lint — configuración

```yaml
# .golangci.yml
run:
  timeout: 5m
  go: "1.22"

linters:
  enable:
    - errcheck # detecta errores ignorados
    - gosimple # simplificaciones de código
    - govet # análisis estático oficial
    - ineffassign # asignaciones inútiles
    - staticcheck # análisis avanzado
    - unused # código no usado
    - goimports # imports ordenados
    - gocritic # sugerencias de mejora
    - revive # reemplaza golint
    - gosec # vulnerabilidades de seguridad
    - bodyclose # detecta response body sin cerrar
    - noctx # detecta HTTP requests sin context
    - prealloc # detecta slices que podrían pre-alocalrse

linters-settings:
  govet:
    enable-all: true
  gosec:
    excludes:
      - G107 # URL de variable (false positive en testing)
  revive:
    rules:
      - name: exported # documentación de tipos exportados

issues:
  exclude-rules:
    # Los archivos de test tienen reglas más laxas
    - path: _test\.go
      linters:
        - gosec
        - errcheck
```

---

## Makefile — comandos comunes

```makefile
.PHONY: test lint build docker-build docker-run migrate

# Variables
BINARY    := server
VERSION   := $(shell git rev-parse --short HEAD)
DOCKER_IMAGE := gestion_productos

# Build
build:
	CGO_ENABLED=0 go build -ldflags="-s -w -X main.version=$(VERSION)" -o $(BINARY) ./cmd/main.go

# Tests con race y cobertura
test:
	go test -race -coverprofile=coverage.out ./...
	go tool cover -func=coverage.out

# Tests de integración
test-integration:
	go test -race -tags integration ./...

# Lint
lint:
	golangci-lint run ./...

# Docker
docker-build:
	docker build -t $(DOCKER_IMAGE):$(VERSION) .

docker-run:
	docker compose up --build

# Migraciones
migrate-up:
	migrate -path ./migrations -database "$(DATABASE_URL)" up

migrate-down:
	migrate -path ./migrations -database "$(DATABASE_URL)" down 1

# Generar código GraphQL / mocks
generate:
	go generate ./...

# Limpiar
clean:
	rm -f $(BINARY) coverage.out
```

---

## Práctica: Novato vs Profesional

### Novato

```yaml
# Solo testea, sin lint, sin cobertura mínima
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test ./... # sin -race, sin cobertura
```

### Profesional

```
- lint: golangci-lint con gosec (seguridad)
- test: -race -coverprofile, umbral mínimo de cobertura
- build: CGO_ENABLED=0, -ldflags="-s -w", -trimpath
- docker: multi-stage, cache de layers, publicar en registry
- secrets: GITHUB_TOKEN automático, nunca hardcodeados
- jobs dependientes: test fallido = no se publica Docker
```
