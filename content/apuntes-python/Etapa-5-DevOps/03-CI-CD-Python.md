# CI/CD con GitHub Actions — Python

La guía de Go tiene `02-CI-CD.md`. Este archivo cubre lo mismo para proyectos Python con FastAPI, pytest, Docker y despliegue a un servidor.

CI = cada push corre tests y lint automáticamente.  
CD = si todo pasa en `main`, se despliega solo.

---

## Estructura del workflow

```
.github/
└── workflows/
    ├── ci.yml       # corre en todo push/PR
    └── deploy.yml   # corre solo en push a main
```

---

## `ci.yml` — Lint, typecheck y tests

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      # Base de datos para los tests de integración
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Instalar uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true # cachea el virtualenv entre runs — mucho más rápido

      - name: Instalar dependencias
        run: uv sync --frozen

      - name: Lint con ruff
        run: uv run ruff check .

      - name: Formato con ruff
        run: uv run ruff format --check .

      - name: Typecheck con mypy
        run: uv run mypy src/

      - name: Tests con pytest
        env:
          DATABASE_URL: postgresql+asyncpg://test_user:test_password@localhost:5432/test_db
          SECRET_KEY: test-secret-key-for-ci
          ENVIRONMENT: test
        run: |
          uv run pytest tests/ -v \
            --cov=src \
            --cov-report=term-missing \
            --cov-fail-under=80
```

---

## `deploy.yml` — Build y despliegue

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    # Solo correr si los tests pasaron
    needs: [] # si ci.yml es un workflow separado, esto no aplica automáticamente

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Login a Docker Hub (o ghcr.io)
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build y push imagen Docker
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            mi-usuario/mi-app:latest
            mi-usuario/mi-app:${{ github.sha }}

      - name: Desplegar en el servidor vía SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mi-app
            docker compose pull
            docker compose up -d --no-build
            docker compose exec -T app alembic upgrade head
            docker system prune -f    # limpiar imágenes viejas
```

---

## Workflow combinado (más simple para proyectos pequeños)

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    name: Lint y Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: ci
          POSTGRES_PASSWORD: ci
          POSTGRES_DB: ci
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run mypy src/
      - run: uv run pytest tests/ --cov=src --cov-fail-under=80
        env:
          DATABASE_URL: postgresql+asyncpg://ci:ci@localhost:5432/ci
          ENVIRONMENT: test

  cd:
    name: Deploy
    runs-on: ubuntu-latest
    needs: ci # solo si CI pasó
    if: github.ref == 'refs/heads/main' # solo en push a main, no en PRs

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: mi-usuario/mi-app:${{ github.sha }},mi-usuario/mi-app:latest

      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mi-app
            echo "IMAGE_TAG=${{ github.sha }}" > .env.deploy
            docker compose pull
            docker compose up -d
```

---

## Secrets en GitHub Actions

En Settings → Secrets and variables → Actions:

| Secret            | Descripción                                     |
| ----------------- | ----------------------------------------------- |
| `DOCKER_USERNAME` | Usuario de Docker Hub                           |
| `DOCKER_TOKEN`    | Access token de Docker Hub (no la contraseña)   |
| `SERVER_HOST`     | IP o dominio del servidor                       |
| `SERVER_USER`     | Usuario SSH (ej: `deploy`)                      |
| `SSH_PRIVATE_KEY` | Clave privada SSH para conectarse al servidor   |
| `DATABASE_URL`    | URL de producción (si la app la necesita en CI) |

Los secrets nunca aparecen en los logs.

---

## Configurar el servidor para recibir despliegues

```bash
# En el servidor de producción
# Crear usuario con permisos mínimos para deploys
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy

# Agregar la clave pública de GitHub Actions al servidor
sudo -u deploy mkdir -p /home/deploy/.ssh
# Pegar el contenido de la clave pública:
sudo -u deploy nano /home/deploy/.ssh/authorized_keys
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys

# Crear directorio de la app
sudo mkdir -p /opt/mi-app
sudo chown deploy:deploy /opt/mi-app

# Copiar docker-compose.yml y .env al servidor (una sola vez)
scp docker-compose.yml deploy@servidor:/opt/mi-app/
scp .env.production deploy@servidor:/opt/mi-app/.env
```

---

## Cache para acelerar los workflows

```yaml
# uv cachea automáticamente con astral-sh/setup-uv@v3 y enable-cache: true
# Para Docker layers con buildx:
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha # usar GitHub Actions cache para Docker layers
    cache-to: type=gha,mode=max
    push: true
    tags: mi-usuario/mi-app:latest
```

---

## Badges de estado en el README

```markdown
[![CI](https://github.com/tu-usuario/mi-repo/actions/workflows/ci-cd.yml/badge.svg)](https://github.com/tu-usuario/mi-repo/actions/workflows/ci-cd.yml)
```

---

## Debugging cuando falla el pipeline

```bash
# Ver logs de un workflow fallido
# GitHub → Actions → seleccionar el run → expandir el step fallido

# Re-run solo los jobs fallidos (sin correr todo de nuevo)
# GitHub → Actions → Re-run failed jobs

# Agregar debug temporalmente
- name: Debug entorno
  run: |
    env | sort           # ver todas las variables de entorno disponibles
    python --version
    uv run pip list      # ver dependencias instaladas
```
