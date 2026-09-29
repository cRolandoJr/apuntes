[[0. Índice DevOps]]

# CI/CD con GitHub Actions

## 1. ¿Qué es CI/CD?

```
CI (Continuous Integration):
  Cada push dispara: build → test → lint
  Si falla, te enterás inmediatamente

CD (Continuous Delivery):
  Después del CI, se despliega automáticamente a staging/producción

Pipeline típico:
  Push → Build → Test → Lint → Build Image → Deploy
```

---

## 2. Estructura de GitHub Actions

Los workflows van en `.github/workflows/` dentro de tu repo.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# ¿Cuándo se ejecuta?
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# ¿Qué hace?
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: Instalar dependencias
        run: go mod download

      - name: Ejecutar tests
        run: go test ./...

      - name: Build
        run: go build -o app ./cmd/main.go
```

---

## 3. Triggers (on:)

```yaml
# Push a branches específicas
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'        # Solo si cambian archivos en src/
      - '!*.md'         # Ignorar cambios en markdown

# Pull requests
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

# Manual (botón en GitHub)
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Ambiente para deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

# Programado (cron)
on:
  schedule:
    - cron: '0 6 * * 1'    # Lunes a las 6 AM UTC

# Tags (releases)
on:
  push:
    tags:
      - 'v*.*.*'
```

---

## 4. Jobs y Steps

```yaml
jobs:
  # Job 1: Lint y test
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        run: |
          echo "Ejecutando linter..."
          golangci-lint run

      - name: Test
        run: go test -v -coverprofile=coverage.out ./...

      - name: Coverage
        run: go tool cover -func=coverage.out

  # Job 2: Build (depende de lint-and-test)
  build:
    runs-on: ubuntu-latest
    needs: lint-and-test # Solo si lint-and-test pasa
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: go build -o app ./cmd/main.go

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-binary
          path: app

  # Job 3: Deploy (solo en main)
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

### Matriz (testear en múltiples versiones)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: ["1.21", "1.22", "1.23"]
        os: [ubuntu-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
      - run: go test ./...
```

---

## 5. Variables y Secrets

```yaml
# Variables de entorno
env:
  APP_NAME: mi-app
  GO_VERSION: "1.22"

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      BUILD_ENV: production

    steps:
      - name: Usar variables
        run: |
          echo "App: $APP_NAME"
          echo "Env: $BUILD_ENV"

      # Usar secrets (configurados en GitHub → Settings → Secrets)
      - name: Login a Docker Hub
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      # Variables de GitHub (automáticas)
      - name: Info del commit
        run: |
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Repo: ${{ github.repository }}"
```

---

## 6. Docker Build y Push

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Metadata (tags y labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: miusuario/mi-app
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build y Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 7. Deploy por SSH

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: build
  if: github.ref == 'refs/heads/main'

  steps:
    - name: Deploy al servidor
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SERVER_SSH_KEY }}
        port: ${{ secrets.SERVER_PORT }}
        script: |
          cd /opt/mi-app
          docker compose pull
          docker compose up -d
          docker image prune -f
```

---

## 8. Servicios (bases de datos para tests)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: go test ./...
```

---

## 9. Caché (Acelerar Pipelines)

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: actions/setup-go@v5
    with:
      go-version: "1.22"
      cache: true # Cache automático para Go

  # Cache manual (para otros casos)
  - name: Cache de dependencias
    uses: actions/cache@v4
    with:
      path: |
        ~/.cache/go-build
        ~/go/pkg/mod
      key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
      restore-keys: |
        ${{ runner.os }}-go-
```

---

## 10. Workflow Reutilizable

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
    secrets:
      SERVER_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy ${{ inputs.environment }}
        run: echo "Deploying ${{ inputs.image_tag }} to ${{ inputs.environment }}"
```

```yaml
# .github/workflows/main.yml — Llamar al workflow reutilizable
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image_tag: ${{ github.sha }}
    secrets:
      SERVER_KEY: ${{ secrets.STAGING_KEY }}

  deploy-prod:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image_tag: ${{ github.sha }}
    secrets:
      SERVER_KEY: ${{ secrets.PROD_KEY }}
```

---

## 11. Pipeline Completo de Ejemplo

```yaml
# .github/workflows/pipeline.yml
name: Full Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: docker.io
  IMAGE_NAME: miusuario/mi-app

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.22" }
      - name: Lint
        uses: golangci/golangci-lint-action@v4

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb }
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.22", cache: true }
      - run: go test -race -coverprofile=coverage.out ./...
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/testdb

  build-push:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }},${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/mi-app
            docker compose pull
            docker compose up -d --remove-orphans
```
