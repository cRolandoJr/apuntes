[[0. Índice DevOps]]

# Secrets Management

## 1. El Problema

```
MAL:
  - Passwords hardcodeados en el código
  - Secrets en archivos .env commiteados a Git
  - Secrets como variables de entorno del CI sin encriptar
  - Passwords compartidos por Slack/WhatsApp

BIEN:
  - Secrets almacenados en un sistema dedicado
  - Acceso controlado por identidad (quién puede leer qué)
  - Rotación automática
  - Auditoría (quién accedió a qué, cuándo)
```

---

## 2. Niveles de Madurez

```
Nivel 0: Secrets en el código o .env (inaceptable)
Nivel 1: Secrets en variables del CI/CD (GitHub Secrets, GitLab CI Variables)
Nivel 2: Secrets encriptados en Git (Sealed Secrets, SOPS)
Nivel 3: Secrets manager externo (Vault, AWS Secrets Manager)
Nivel 4: Secrets dinámicos + rotación automática (Vault dynamic secrets)
```

---

## 3. GitHub Secrets (Nivel 1)

Para CI/CD pipelines.

```
GitHub → Repo → Settings → Secrets and variables → Actions → New repository secret
```

```yaml
# Usar en GitHub Actions
jobs:
  deploy:
    steps:
      - name: Login Docker
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Deploy
        env:
          SERVER_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: ./deploy.sh
```

> Limitación: Solo accesibles en pipelines. No sirven para apps en runtime.

---

## 4. Sealed Secrets (Nivel 2 — Kubernetes)

Encriptar secrets para guardarlo en Git. Solo el cluster puede desencriptarlo.

```bash
# Instalar controlador en el cluster
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Instalar CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz
tar xzf kubeseal-*.tar.gz
sudo install kubeseal /usr/local/bin/
```

```bash
# Crear un Secret normal
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=mi-password-seguro \
  --dry-run=client -o yaml > secret.yaml

# Encriptarlo (Sealed Secret)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# El archivo sealed-secret.yaml es SEGURO para commitear a Git
cat sealed-secret.yaml
```

```yaml
# sealed-secret.yaml (seguro en Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
spec:
  encryptedData:
    username: AgBy3i... # Encriptado
    password: AgCtr8... # Encriptado
```

```bash
# Aplicar (el controlador lo desencripta automáticamente)
kubectl apply -f sealed-secret.yaml
# Crea un Secret normal que los pods pueden usar
```

---

## 5. SOPS (Nivel 2 — Multi-plataforma)

Mozilla SOPS encripta valores en archivos YAML/JSON, dejando las keys visibles.

```bash
# Instalar
wget https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
sudo install sops-* /usr/local/bin/sops

# Crear key con age (más simple que GPG)
age-keygen -o keys.txt
# Guardar la clave pública
```

```bash
# Encriptar archivo
sops --encrypt --age age1ql3z7hjy... secrets.yaml > secrets.enc.yaml

# Editar archivo encriptado (se desencripta temporalmente)
sops secrets.enc.yaml

# Desencriptar
export SOPS_AGE_KEY_FILE=./keys.txt
sops --decrypt secrets.enc.yaml
```

```yaml
# secrets.enc.yaml (seguro en Git)
# Las KEYS son visibles, los VALORES están encriptados
db:
  host: ENC[AES256_GCM,data:4kR3...]
  password: ENC[AES256_GCM,data:9xZp...]
  port: ENC[AES256_GCM,data:J2...]
sops:
  age:
    - recipient: age1ql3z7hjy...
```

### SOPS con Terraform

```hcl
# Leer secrets encriptados con SOPS
data "sops_file" "secrets" {
  source_file = "secrets.enc.yaml"
}

resource "aws_db_instance" "main" {
  password = data.sops_file.secrets.data["db.password"]
}
```

---

## 6. HashiCorp Vault (Nivel 3-4)

El estándar de la industria para secrets management.

### Instalación (Dev Mode para practicar)

```bash
# Docker
docker run -d --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=mi-root-token \
  hashicorp/vault

# CLI
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='mi-root-token'
```

### Operaciones Básicas

```bash
# Guardar un secret
vault kv put secret/mi-app/db \
  username=admin \
  password=password-seguro \
  host=db.ejemplo.com

# Leer un secret
vault kv get secret/mi-app/db
vault kv get -field=password secret/mi-app/db

# Listar secrets
vault kv list secret/mi-app/

# Borrar
vault kv delete secret/mi-app/db

# Versiones (Vault mantiene historial)
vault kv get -version=1 secret/mi-app/db
```

### Políticas (Quién puede acceder a qué)

```hcl
# policy-dev.hcl
# Los developers solo pueden leer secrets de dev
path "secret/data/dev/*" {
  capabilities = ["read", "list"]
}

# No acceso a producción
path "secret/data/prod/*" {
  capabilities = ["deny"]
}
```

```bash
# Crear política
vault policy write dev-policy policy-dev.hcl

# Crear token con esa política
vault token create -policy=dev-policy
```

### Auth Methods (Cómo autenticarse)

```bash
# Kubernetes auth (pods se autentican automáticamente)
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# AppRole (para aplicaciones)
vault auth enable approle
vault write auth/approle/role/mi-app \
  token_policies="app-policy" \
  token_ttl=1h

# El app obtiene un token:
vault write auth/approle/login \
  role_id="xxx" \
  secret_id="yyy"
```

### En Kubernetes (External Secrets Operator)

```bash
# Instalar External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

```yaml
# ClusterSecretStore (conexión a Vault)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.ejemplo.com"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "mi-app"
---
# ExternalSecret (sincronizar un secret de Vault a K8s)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: username
      remoteRef:
        key: secret/mi-app/db
        property: username
    - secretKey: password
      remoteRef:
        key: secret/mi-app/db
        property: password
```

---

## 7. AWS Secrets Manager

```bash
# Crear secret
aws secretsmanager create-secret \
  --name mi-app/db-password \
  --secret-string "password-seguro"

# Leer
aws secretsmanager get-secret-value --secret-id mi-app/db-password

# Rotar (configurar rotación automática)
aws secretsmanager rotate-secret --secret-id mi-app/db-password
```

```hcl
# Terraform
resource "aws_secretsmanager_secret" "db_password" {
  name = "mi-app/db-password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = var.db_password
}
```

---

## 8. Buenas Prácticas

```
1. NUNCA commitear secrets a Git (ni siquiera en ramas privadas)
2. Usar .gitignore: .env, *.pem, *.key, terraform.tfvars
3. Pre-commit hook para detectar secrets:
   - git-secrets (AWS)
   - gitleaks
   - detect-secrets
4. Rotar secrets periódicamente
5. Principio de mínimo privilegio (cada app solo accede a SUS secrets)
6. Auditar accesos a secrets
7. Usar secrets dinámicos cuando sea posible (Vault genera credenciales temporales)
8. En K8s: External Secrets Operator > Sealed Secrets > Secret YAML en Git
```

### Pre-commit para detectar secrets

```bash
# Instalar gitleaks
brew install gitleaks   # o descagar binario

# Escanear repo
gitleaks detect --source .

# Como pre-commit hook
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```
