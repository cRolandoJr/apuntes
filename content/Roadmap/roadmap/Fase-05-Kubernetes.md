# Fase 5: Kubernetes & Orquestación — Meses 13 a 15

> **Objetivo**: Entender Kubernetes de arriba a abajo — no solo copiar YAMLs, sino saber qué hace cada componente y por qué. Al terminar, debés ser capaz de deployar, escalar, debuggear y mantener aplicaciones en K8s. Preparación para el CKA.

---

## Mes 13: Arquitectura y Conceptos Fundamentales

### 13.1 ¿Por qué Kubernetes?

Docker Compose funciona para una máquina. Pero en producción real:

- ¿Qué pasa si la máquina muere? → Necesitás **alta disponibilidad**
- ¿Qué si un servicio necesita más instancias? → **Escalado automático**
- ¿Cómo balanceás tráfico entre copias? → **Service discovery + load balancing**
- ¿Cómo hacés deploys sin downtime? → **Rolling updates**
- ¿Cómo gestionás configuración y secretos? → **ConfigMaps y Secrets**

Kubernetes (K8s) resuelve todo esto.

### 13.2 Arquitectura de un Cluster

```
┌──────────────── Control Plane ────────────────┐
│                                                │
│  ┌─────────────┐  ┌──────────────┐            │
│  │ API Server  │  │   etcd       │            │
│  │ (kube-api)  │  │ (base datos) │            │
│  └──────┬──────┘  └──────────────┘            │
│         │                                      │
│  ┌──────┴──────┐  ┌──────────────────┐        │
│  │ Scheduler   │  │ Controller       │        │
│  │ (dónde poner│  │ Manager          │        │
│  │  cada pod)  │  │ (estado deseado) │        │
│  └─────────────┘  └──────────────────┘        │
└────────────────────────────────────────────────┘
              │
              │  kubectl / API
              │
┌─────────────┴──── Worker Nodes ───────────────┐
│                                                │
│  Node 1                    Node 2              │
│  ┌──────────────────┐     ┌─────────────────┐ │
│  │ kubelet          │     │ kubelet         │ │
│  │ kube-proxy       │     │ kube-proxy      │ │
│  │ container runtime│     │ container runtime│ │
│  │                  │     │                 │ │
│  │ ┌─Pod─┐ ┌─Pod─┐ │     │ ┌─Pod─┐        │ │
│  │ │App A│ │App B│ │     │ │App A│        │ │
│  │ └─────┘ └─────┘ │     │ └─────┘        │ │
│  └──────────────────┘     └─────────────────┘ │
└────────────────────────────────────────────────┘
```

**Componentes del Control Plane**:

- **API Server**: El "front desk" del cluster. TODO pasa por aquí (kubectl, controllers, etc.)
- **etcd**: Base de datos key-value. Guarda el estado de TODO el cluster. Si se pierde, se pierde el cluster.
- **Scheduler**: Decide en qué nodo poner cada Pod nuevo (basado en recursos disponibles, afinidad, etc.)
- **Controller Manager**: Vigila que el estado real coincida con el estado deseado. "Dijiste 3 réplicas, solo hay 2 → creo una más."

**Componentes de cada Worker Node**:

- **kubelet**: Agente que corre en cada nodo. Recibe instrucciones del API Server y gestiona los Pods.
- **kube-proxy**: Maneja las reglas de red — cómo el tráfico llega a los Pods correctos.
- **Container Runtime**: Lo que ejecuta los containers (containerd, CRI-O).

### 13.3 El Modelo Declarativo

K8s es **declarativo**: vos le decís "quiero 3 réplicas de mi app" y K8s se encarga de que SIEMPRE haya 3. Si una muere, crea otra. Si sobra una, la mata.

```yaml
# "Yo quiero esto":
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3 # Siempre 3 copias
```

K8s constantemente compara el **estado deseado** (lo que declaraste) con el **estado actual** (lo que hay corriendo) y actúa para que coincidan. Esto es un **reconciliation loop**.

### 13.4 Los Objetos Fundamentales

**Pod**: La unidad mínima. Uno o más containers que comparten red y almacenamiento.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-api
  labels:
    app: gestion-productos
spec:
  containers:
    - name: api
      image: gestion-productos:v1.0.0
      ports:
        - containerPort: 8081
      resources:
        requests: # Mínimo garantizado
          cpu: "100m" # 100 milicores = 0.1 CPU
          memory: "64Mi"
        limits: # Máximo permitido
          cpu: "500m"
          memory: "256Mi"
      livenessProbe: # ¿Está vivo? Si falla, K8s reinicia el container
        httpGet:
          path: /
          port: 8081
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe: # ¿Puede recibir tráfico? Si falla, se saca del Service
        httpGet:
          path: /
          port: 8081
        initialDelaySeconds: 3
        periodSeconds: 5
```

**Deployment**: Gestiona Pods con réplicas, rolling updates y rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gestion-productos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gestion-productos # ¿Qué Pods gestiono? Los que tengan este label
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # Crear máximo 1 pod nuevo antes de matar uno viejo
      maxUnavailable: 0 # Siempre tener todos disponibles
  template:
    metadata:
      labels:
        app: gestion-productos
    spec:
      containers:
        - name: api
          image: gestion-productos:v1.0.0
          ports:
            - containerPort: 8081
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

**Service**: Expone Pods como un endpoint estable de red.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gestion-productos-svc
spec:
  selector:
    app: gestion-productos # ¿A qué pods mando tráfico?
  ports:
    - port: 80 # Puerto del Service
      targetPort: 8081 # Puerto del container
  type: ClusterIP # Solo accesible dentro del cluster
  # Tipos:
  # ClusterIP  → Interno. Otros Pods lo alcanzan por nombre.
  # NodePort   → Expone en un puerto de cada nodo (30000-32767)
  # LoadBalancer → Crea un load balancer externo (cloud)
```

**ConfigMap y Secret**:

```yaml
# ConfigMap: configuración no sensible
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  PORT: "8081"
  LOG_LEVEL: "info"
  DB_HOST: "postgres-svc"

---
# Secret: datos sensibles (codificados en base64, NO encriptados)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM= # echo -n "password123" | base64
```

```yaml
# Usar en un Deployment:
spec:
  containers:
    - name: api
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-credentials
```

**Ingress**: Ruteo HTTP externo → Services internos.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gestion-productos-svc
                port:
                  number: 80
```

### 13.5 Setup del Lab Local

```bash
# Opción 1: minikube (recomendado para aprender)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --cpus=2 --memory=4096

# Opción 2: k3s (más liviano, más "real")
curl -sfL https://get.k3s.io | sh -

# kubectl (cliente de K8s)
# Si usás minikube:
alias kubectl="minikube kubectl --"
# O instalá kubectl directamente:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# Verificar
kubectl cluster-info
kubectl get nodes
```

---

## Mes 14: Operaciones del Día a Día

### 14.1 kubectl — Los Comandos Esenciales

```bash
# Información del cluster
kubectl cluster-info
kubectl get nodes -o wide

# Listar recursos
kubectl get pods                    # Pods en el namespace actual
kubectl get pods -A                 # Todos los namespaces
kubectl get pods -o wide            # Más info (nodo, IP)
kubectl get deployments
kubectl get services
kubectl get all                     # Todo junto

# Describir un recurso (debug)
kubectl describe pod <nombre>
kubectl describe deployment <nombre>
# Mirá la sección Events al final — ahí están los errores

# Logs
kubectl logs <pod>                  # Logs del container
kubectl logs <pod> -f               # Follow
kubectl logs <pod> --previous       # Logs del container anterior (si crasheó)
kubectl logs -l app=gestion-productos  # Logs por label

# Ejecutar comandos dentro de un Pod
kubectl exec -it <pod> -- sh       # Shell interactivo
kubectl exec <pod> -- cat /etc/hosts

# Port forward (acceder a un servicio sin exponerlo)
kubectl port-forward svc/gestion-productos-svc 8081:80
# Ahora localhost:8081 llega al Service

# Aplicar configuración
kubectl apply -f deployment.yaml   # Crear o actualizar
kubectl delete -f deployment.yaml  # Eliminar

# Escalar
kubectl scale deployment gestion-productos --replicas=5

# Rolling update
kubectl set image deployment/gestion-productos api=gestion-productos:v2.0.0
kubectl rollout status deployment/gestion-productos   # Seguir progreso
kubectl rollout history deployment/gestion-productos   # Historial
kubectl rollout undo deployment/gestion-productos      # Rollback!

# Debug: ¿por qué mi pod no arranca?
kubectl get pods
# Si dice "ImagePullBackOff" → no encuentra la imagen
# Si dice "CrashLoopBackOff" → el container arranca y muere repetidamente
# Si dice "Pending" → no hay nodo con recursos suficientes
kubectl describe pod <pod>   # Los Events dicen exactamente qué pasó
kubectl logs <pod>           # Si crasheó, los logs dicen por qué
```

### 14.2 Namespaces

Namespaces separan recursos dentro del cluster (como "carpetas" lógicas).

```bash
kubectl create namespace staging
kubectl create namespace production

# Deployar en un namespace específico
kubectl apply -f deployment.yaml -n staging

# Ver recursos de un namespace
kubectl get pods -n staging
kubectl get all -n production

# Cambiar namespace default de kubectl
kubectl config set-context --current --namespace=staging
```

### 14.3 Helm — El Package Manager de K8s

Helm es como `apt` pero para Kubernetes. En vez de escribir 10 archivos YAML, usás un "chart".

```bash
# Instalar Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Agregar repositorios
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Instalar PostgreSQL con Helm
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=secreto123 \
  --set primary.persistence.size=10Gi \
  -n production

# Ver releases instaladas
helm list -A

# Ver valores configurables de un chart
helm show values bitnami/postgresql | head -50

# Actualizar una release
helm upgrade my-postgres bitnami/postgresql \
  --set auth.postgresPassword=secreto123 \
  --set primary.resources.limits.memory=1Gi

# Desinstalar
helm uninstall my-postgres -n production

# Crear tu propio chart
helm create gestion-productos
# Genera:
# gestion-productos/
# ├── Chart.yaml           → Metadata
# ├── values.yaml           → Valores configurables
# └── templates/
#     ├── deployment.yaml   → Template con {{ .Values.xxx }}
#     ├── service.yaml
#     └── ingress.yaml
```

---

## Mes 15: Temas Avanzados y CKA

### 15.1 Persistent Volumes

```yaml
# PersistentVolumeClaim: "Necesito 10GB de disco"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce # Un solo nodo puede montar para escritura
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard # El provisioner asigna automáticamente

---
# Usar en un Pod:
spec:
  containers:
    - name: postgres
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data
```

### 15.2 Network Policies (Firewall de K8s)

```yaml
# Solo permitir tráfico de la API al PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-policy
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: gestion-productos # Solo estos pods pueden acceder
      ports:
        - port: 5432
```

### 15.3 RBAC (Role-Based Access Control)

```yaml
# Role: qué se puede hacer
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding: quién puede hacerlo
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: rolando
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 15.4 HPA (Horizontal Pod Autoscaler)

```yaml
# Escalar automáticamente basado en CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gestion-productos
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70 # Escalar cuando CPU > 70%
```

---

## Proyecto Integrador de Fase 5

### "Stack Completo en Kubernetes"

Deployar tu API de productos en un cluster local (minikube o k3s):

1. **Deployment** de la API (3 réplicas, probes, resources)
2. **Service** ClusterIP
3. **Ingress** para acceso HTTP externo
4. **PostgreSQL** con Helm (PVC para datos persistentes)
5. **ConfigMap** para configuración de la app
6. **Secret** para credenciales de BD
7. **NetworkPolicy**: Solo la API puede hablar con PostgreSQL
8. **HPA** que escale entre 2 y 5 réplicas por CPU

**Verificación**:

```bash
kubectl get all -n productos
# Todos los pods Running y Ready

kubectl get pods -n productos
# 3 réplicas de la API, 1 de PostgreSQL

kubectl exec -it deploy/gestion-productos -n productos -- wget -qO- http://localhost:8081/
# La API responde desde dentro del cluster

# Acceso externo (minikube)
minikube service gestion-productos-svc -n productos
# O con ingress:
curl http://api.local/query -X POST -H "Content-Type: application/json" \
  -d '{"query": "{ products { id name } }"}'

# Ver escalado automático
kubectl get hpa -n productos
kubectl top pods -n productos

# Simular fallo
kubectl delete pod <un-pod> -n productos
kubectl get pods -n productos -w
# K8s crea uno nuevo automáticamente
```

---

## Recursos para esta Fase

### Kubernetes

1. **Kubernetes Official Docs** (kubernetes.io/docs) — La biblia. Completa y con ejemplos.
2. **"Kubernetes in Action" de Marko Lukša** (2da edición) — EL libro. Profundo y claro.
3. **"Kubernetes Up & Running" de Kelsey Hightower** — Más conciso, excelente para empezar.
4. **Kubernetes The Hard Way** (github.com/kelseyhightower/kubernetes-the-hard-way) — Construir K8s desde cero para entender cada pieza.

### Preparación CKA

- **KillerCoda CKA** (killercoda.com) — Labs gratuitos enfocados en CKA
- **killer.sh** — Simulador de examen CKA (viene gratis con la inscripción al exam)
- **CKA Curriculum** (github.com/cncf/curriculum) — Los temas oficiales

### Helm

- **Helm Docs** (helm.sh/docs) — Documentación oficial
- **Artifact Hub** (artifacthub.io) — Buscar charts públicos

---

## Checkpoint / CKA Readiness

- [ ] ¿Puedo explicar la arquitectura del control plane?
- [ ] ¿Puedo crear Deployments, Services, Ingress desde cero?
- [ ] ¿Puedo debuggear un Pod que no arranca?
- [ ] ¿Puedo hacer rolling updates y rollbacks?
- [ ] ¿Puedo usar ConfigMaps y Secrets?
- [ ] ¿Puedo crear Network Policies?
- [ ] ¿Puedo configurar RBAC?
- [ ] ¿Puedo usar Helm para instalar y gestionar aplicaciones?
- [ ] ¿Puedo configurar Persistent Volumes?
- [ ] ¿Puedo configurar un HPA?

**Si querés rendir el CKA, hacelo al final de esta fase.** Es un examen práctico en terminal — todo lo que practicaste aplica directamente.
