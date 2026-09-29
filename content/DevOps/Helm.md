[[0. Índice DevOps]]

# Helm — Package Manager para Kubernetes

## 1. ¿Qué es Helm?

Helm es el "apt/dnf de Kubernetes". En vez de escribir 15 archivos YAML para deployar una app, usás un **chart** (paquete) que agrupa todo con valores configurables.

```
Sin Helm:
  kubectl apply -f namespace.yml
  kubectl apply -f configmap.yml
  kubectl apply -f secret.yml
  kubectl apply -f deployment.yml
  kubectl apply -f service.yml
  kubectl apply -f ingress.yml
  (y repetir para cada ambiente cambiando valores a mano)

Con Helm:
  helm install mi-app ./mi-chart -f valores-prod.yml
```

---

## 2. Instalación

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 3. Usar Charts de la Comunidad

```bash
# Agregar repositorio
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Buscar charts
helm search repo nginx
helm search repo postgres

# Ver valores configurables de un chart
helm show values bitnami/postgresql

# Instalar
helm install mi-postgres bitnami/postgresql \
  --namespace databases \
  --create-namespace \
  --set auth.postgresPassword=mi-password \
  --set primary.persistence.size=10Gi

# Instalar con archivo de valores
helm install mi-postgres bitnami/postgresql \
  -n databases --create-namespace \
  -f postgres-values.yml

# Ver releases instalados
helm list
helm list -A    # Todos los namespaces

# Ver estado
helm status mi-postgres

# Actualizar valores
helm upgrade mi-postgres bitnami/postgresql \
  -n databases -f postgres-values.yml

# Rollback (OJO: es `helm history`, NO `helm rollout` — rollout es de kubectl)
helm history mi-postgres
helm rollback mi-postgres 1

# Desinstalar
helm uninstall mi-postgres -n databases
```

---

## 4. Crear tu Propio Chart

```bash
# Generar estructura
helm create mi-app
```

```
mi-app/
├── Chart.yaml           # Metadata del chart
├── values.yaml          # Valores por defecto
├── templates/           # Templates de K8s
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl     # Funciones helper
│   └── NOTES.txt        # Mensaje post-install
└── charts/              # Dependencias
```

### Chart.yaml

```yaml
apiVersion: v2
name: mi-app
description: Mi aplicación web
type: application
version: 0.1.0 # Versión del chart
appVersion: "1.0.0" # Versión de la app

dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
# values.yaml — Valores por defecto
replicaCount: 2

image:
  repository: miusuario/mi-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  host: app.ejemplo.com
  tls: true

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

env:
  APP_ENV: production
  LOG_LEVEL: info

postgresql:
  enabled: true
  auth:
    postgresPassword: "" # Se define por ambiente

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 80
```

### Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mi-app.fullname" . }}
  labels:
    {{- include "mi-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mi-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mi-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
```

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: { { include "mi-app.fullname" . } }
spec:
  type: { { .Values.service.type } }
  selector: { { - include "mi-app.selectorLabels" . | nindent 4 } }
  ports:
    - port: { { .Values.service.port } }
      targetPort: 8080
```

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mi-app.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "mi-app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

---

## 5. Valores por Ambiente

```yaml
# values-dev.yml
replicaCount: 1
image:
  tag: "dev-latest"
ingress:
  host: dev.app.ejemplo.com
resources:
  requests:
    cpu: 50m
    memory: 64Mi
env:
  APP_ENV: development
  LOG_LEVEL: debug
```

```yaml
# values-prod.yml
replicaCount: 3
image:
  tag: "v1.2.3"
ingress:
  host: app.ejemplo.com
  tls: true
resources:
  requests:
    cpu: 200m
    memory: 256Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
env:
  APP_ENV: production
  LOG_LEVEL: warn
```

```bash
# Deploy a dev
helm upgrade --install mi-app ./mi-app \
  -n development --create-namespace \
  -f values-dev.yml

# Deploy a prod
helm upgrade --install mi-app ./mi-app \
  -n production --create-namespace \
  -f values-prod.yml
```

---

## 6. Comandos Útiles

```bash
# Validar chart
helm lint ./mi-app

# Render templates sin instalar (ver el YAML final)
helm template mi-app ./mi-app -f values-prod.yml

# Dry run
helm install mi-app ./mi-app --dry-run --debug

# Ver diferencias antes de upgrade
helm diff upgrade mi-app ./mi-app -f values-prod.yml
# (requiere plugin: helm plugin install https://github.com/databus23/helm-diff)

# Ver valores actuales de un release
helm get values mi-app

# Ver todo lo generado
helm get manifest mi-app

# Historial de releases
helm history mi-app

# Empaquetar chart
helm package ./mi-app
# Genera: mi-app-0.1.0.tgz

# Actualizar dependencias
helm dependency update ./mi-app
```

---

## 7. Buenas Prácticas

```
1. Siempre usar values.yaml — nunca hardcodear en templates
2. Un archivo values-*.yml por ambiente (dev, staging, prod)
3. Versionar Chart.yaml y appVersion correctamente
4. helm lint antes de deploy
5. helm template para verificar el YAML generado
6. Usar helm diff antes de upgrade en producción
7. Secrets nunca en values.yaml — usar Sealed Secrets o external-secrets
8. Nombrar recursos con {{ include "chart.fullname" . }}
```
