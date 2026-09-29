[[0. Índice DevOps]]

# GitOps y ArgoCD

## 1. ¿Qué es GitOps?

```
Filosofía: Git es la ÚNICA fuente de verdad para la infraestructura y deployments.

Antes (Push-based):
  CI/CD pipeline → build → kubectl apply (push al cluster)
  Problema: el CI tiene acceso al cluster, el estado real puede divergir

GitOps (Pull-based):
  Git repo define el estado deseado
  ArgoCD dentro del cluster OBSERVA el repo y sincroniza automáticamente
  El cluster se auto-corrige si alguien cambia algo manualmente
```

### Principios

1. **Todo en Git:** Manifiestos K8s, Helm charts, configs
2. **Estado deseado declarativo:** YAML describe qué querés
3. **Cambios via PR:** Modificar infra = Pull Request
4. **Reconciliación automática:** ArgoCD detecta diferencias y corrige

---

## 2. Estructura de Repos

```
# Opción recomendada: 2 repos separados

Repo 1: app-code (código de la app)
├── src/
├── Dockerfile
├── go.mod
└── .github/workflows/ci.yml    # CI: build + push image

Repo 2: app-gitops (manifiestos K8s)
├── base/                       # Manifiestos base
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/                   # Configs por ambiente
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── argocd/
    └── applications.yaml       # Definición de apps en ArgoCD
```

---

## 3. ArgoCD — Instalación

```bash
# Instalar en Kubernetes
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que esté listo
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Obtener password inicial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Acceder al UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Abrir: https://localhost:8080
# User: admin / Password: el de arriba

# CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install argocd /usr/local/bin/argocd

argocd login localhost:8080
argocd account update-password    # Cambiar password
```

---

## 4. Crear una Application

### Via YAML

```yaml
# argocd/mi-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mi-usuario/app-gitops.git
    targetRevision: main
    path: overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: mi-app

  syncPolicy:
    automated:
      prune: true # Borrar recursos que ya no están en Git
      selfHeal: true # Re-sincronizar si alguien modifica manualmente
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
kubectl apply -f argocd/mi-app.yaml
```

### Via CLI

```bash
argocd app create mi-app \
  --repo https://github.com/mi-usuario/app-gitops.git \
  --path overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace mi-app \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

## 5. Kustomize (Personalizar Manifiestos por Ambiente)

ArgoCD soporta Kustomize nativamente.

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: app
          image: miusuario/mi-app:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# `bases:` está deprecado — usar `resources:` (acepta rutas a otras kustomizations)
resources:
  - ../../base
namespace: production

patches:
  - target:
      kind: Deployment
      name: mi-app
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3

images:
  - name: miusuario/mi-app
    newTag: v1.2.3 # El CI actualiza este tag

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=warn
```

```bash
# Previsualizar qué genera kustomize
kubectl kustomize overlays/production/
```

---

## 6. Workflow Completo de GitOps

```
1. Developer pushea código al repo de la app
2. GitHub Actions:
   a. Build + test
   b. Build imagen Docker
   c. Push imagen: miusuario/mi-app:v1.2.3
   d. Actualizar el tag en el repo GitOps (overlays/production/kustomization.yaml)
3. ArgoCD detecta el cambio en el repo GitOps
4. ArgoCD sincroniza: aplica los manifiestos actualizados al cluster
5. Kubernetes hace rolling update con la nueva imagen
```

### CI que actualiza el repo GitOps

```yaml
# En el repo de la app: .github/workflows/ci.yml
update-gitops:
  needs: build-push
  runs-on: ubuntu-latest
  steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: mi-usuario/app-gitops
        token: ${{ secrets.GITOPS_TOKEN }}

    - name: Actualizar tag de imagen
      run: |
        cd overlays/production
        kustomize edit set image miusuario/mi-app=miusuario/mi-app:${{ github.sha }}

    - name: Commit y push
      run: |
        git config user.name "github-actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "deploy: mi-app ${{ github.sha }}"
        git push
```

---

## 7. Comandos de ArgoCD

```bash
# Ver aplicaciones
argocd app list

# Detalle de una app
argocd app get mi-app

# Sincronizar manualmente
argocd app sync mi-app

# Ver diferencias (qué cambiaría)
argocd app diff mi-app

# Historial de syncs
argocd app history mi-app

# Rollback a una revisión anterior
argocd app rollback mi-app 3

# Borrar app (sin borrar recursos del cluster)
argocd app delete mi-app --cascade=false

# Borrar app (Y los recursos del cluster)
argocd app delete mi-app
```

---

## 8. Buenas Prácticas

```
1. Separar repo de código y repo de GitOps
2. Usar Kustomize o Helm para ambientes (overlay por ambiente)
3. Activar auto-sync con self-heal y prune
4. Proteger el branch main del repo GitOps con PR reviews
5. Nunca hacer kubectl apply manual en producción
6. Usar Sealed Secrets o External Secrets para secrets
7. Un Application de ArgoCD por microservicio por ambiente
8. Monitorear ArgoCD con Prometheus (exporta métricas)
```
