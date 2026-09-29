[[0. Índice DevOps]]

# Kubernetes (K8s)

## 1. ¿Qué es Kubernetes?

**Orquestador de contenedores.** Vos le decís "quiero 3 réplicas de mi app" y Kubernetes se encarga de crearlas, mantenerlas vivas, distribuir el tráfico y escalar.

```
Docker = Correr UN contenedor
Docker Compose = Correr VARIOS contenedores en UNA máquina
Kubernetes = Correr MUCHOS contenedores en MUCHAS máquinas
```

### Arquitectura

```
Cluster K8s
├── Control Plane (cerebro)
│   ├── API Server (kubectl habla con esto)
│   ├── etcd (base de datos del cluster)
│   ├── Scheduler (decide dónde correr los pods)
│   └── Controller Manager (mantiene el estado deseado)
│
└── Worker Nodes (músculo)
    ├── kubelet (agente que corre pods)
    ├── kube-proxy (networking)
    └── Container Runtime (containerd / CRI-O)
```

---

## 2. Instalación Local (para practicar)

```bash
# Opción 1: minikube (un cluster de un nodo)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start
minikube status
minikube stop
minikube delete

# Opción 2: kind (Kubernetes in Docker)
go install sigs.k8s.io/kind@latest
kind create cluster
kind delete cluster

# kubectl (CLI para hablar con K8s)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl
kubectl version
```

---

## 3. Conceptos Fundamentales

| Concepto             | Qué es                                                          |
| -------------------- | --------------------------------------------------------------- |
| **Pod**              | Unidad mínima. 1 o más contenedores que comparten red y storage |
| **Deployment**       | Maneja réplicas de pods, rolling updates, rollbacks             |
| **Service**          | Expone pods con una IP/DNS estable                              |
| **Namespace**        | Separación lógica (como carpetas para recursos)                 |
| **ConfigMap**        | Configuración (variables, archivos) NO sensible                 |
| **Secret**           | Configuración sensible (contraseñas, tokens)                    |
| **Ingress**          | Enrutar tráfico HTTP externo a servicios internos               |
| **PersistentVolume** | Almacenamiento persistente                                      |
| **Node**             | Máquina física o virtual del cluster                            |

---

## 4. kubectl — Comandos Esenciales

```bash
# Contexto y cluster
kubectl config get-contexts          # Ver clusters configurados
kubectl config use-context mi-cluster  # Cambiar de cluster
kubectl cluster-info

# Ver recursos
kubectl get pods                     # Pods del namespace actual
kubectl get pods -A                  # Todos los namespaces
kubectl get pods -o wide             # Más info (IP, nodo)
kubectl get deployments
kubectl get services
kubectl get all                      # Todo

# Describir un recurso (debugging)
kubectl describe pod mi-pod
kubectl describe deployment mi-app
kubectl describe node mi-nodo

# Logs
kubectl logs mi-pod
kubectl logs mi-pod -f               # Follow (como tail -f)
kubectl logs mi-pod -c mi-contenedor # Contenedor específico (multi-container pod)
kubectl logs -l app=mi-app           # Por label

# Ejecutar comando en un pod
kubectl exec -it mi-pod -- /bin/sh
kubectl exec mi-pod -- ls /app

# Port forward (acceder a un pod desde tu máquina)
kubectl port-forward pod/mi-pod 8080:80
kubectl port-forward svc/mi-servicio 8080:80

# Aplicar manifiestos YAML
kubectl apply -f deployment.yml
kubectl apply -f ./k8s/              # Todos los YAML de una carpeta

# Borrar recursos
kubectl delete -f deployment.yml
kubectl delete pod mi-pod
kubectl delete deployment mi-app

# Namespaces
kubectl get namespaces
kubectl create namespace staging
kubectl get pods -n staging          # Pods en namespace staging
```

---

## 5. Pod

```yaml
# pod.yml (rara vez se crea solo, normalmente via Deployment)
apiVersion: v1
kind: Pod
metadata:
  name: mi-app
  labels:
    app: mi-app
spec:
  containers:
    - name: app
      image: miusuario/mi-app:v1.0
      ports:
        - containerPort: 8080
      resources:
        requests: # Mínimo garantizado
          memory: "64Mi"
          cpu: "100m" # 100 milicores = 0.1 CPU
        limits: # Máximo permitido
          memory: "128Mi"
          cpu: "250m"
```

---

## 6. Deployment (Lo Más Usado)

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  labels:
    app: mi-app
spec:
  replicas: 3 # 3 instancias corriendo
  selector:
    matchLabels:
      app: mi-app
  template: # Template del Pod
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: app
          image: miusuario/mi-app:v1.0
          ports:
            - containerPort: 8080
          env:
            - name: APP_ENV
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe: # ¿El pod está vivo?
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe: # ¿El pod puede recibir tráfico?
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 # Máximo pods caídos durante update
      maxSurge: 1 # Máximo pods extra durante update
```

```bash
# Gestionar deployments
kubectl apply -f deployment.yml
kubectl get deployments
kubectl rollout status deployment/mi-app

# Escalar
kubectl scale deployment/mi-app --replicas=5

# Actualizar imagen
kubectl set image deployment/mi-app app=miusuario/mi-app:v2.0

# Rollback
kubectl rollout undo deployment/mi-app
kubectl rollout undo deployment/mi-app --to-revision=2
kubectl rollout history deployment/mi-app
```

---

## 7. Service (Exponer la App)

```yaml
# service.yml
apiVersion: v1
kind: Service
metadata:
  name: mi-app-svc
spec:
  selector:
    app: mi-app # Conecta con pods que tengan este label
  ports:
    - port: 80 # Puerto del servicio
      targetPort: 8080 # Puerto del contenedor
  type: ClusterIP # Solo accesible dentro del cluster
```

### Tipos de Service

| Tipo           | Uso                                        |
| -------------- | ------------------------------------------ |
| `ClusterIP`    | Solo accesible internamente (default)      |
| `NodePort`     | Expone en un puerto del nodo (30000-32767) |
| `LoadBalancer` | Crea un load balancer externo (cloud)      |

```yaml
# NodePort (acceso directo al nodo)
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080      # Accesible en http://NODO-IP:30080

# LoadBalancer (cloud)
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  # El cloud provider asigna una IP externa

# ⚠️ EN LAB LOCAL (minikube/kind) `LoadBalancer` queda en <pending> PARA SIEMPRE:
#    no hay cloud que asigne la IP. Alternativas locales:
#    - minikube: `minikube tunnel` (deja corriendo) o usar NodePort
#    - kind: instalar MetalLB, o usar NodePort / port-forward
#    - dev rápido: `kubectl port-forward svc/mi-svc 8080:80`
```

---

## 8. Ingress (Enrutar HTTP)

```yaml
# ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mi-app-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
  tls:
    - hosts:
        - app.ejemplo.com
      secretName: tls-secret
```

```bash
# Instalar Ingress Controller (nginx) — OJO: el provider según dónde corras
# EN CLOUD: usar el manifiesto 'cloud' pineado a un release (NO 'main', que es frágil):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/cloud/deploy.yaml
# EN LOCAL minikube: es más simple el addon →  minikube addons enable ingress
# EN LOCAL kind: usar el provider 'kind' →  .../controller-v1.11.3/deploy/static/provider/kind/deploy.yaml
```

> ⚠️ El manifiesto `provider/cloud` crea un Service `LoadBalancer` → en minikube/kind
> queda `<pending>` y el ingress nunca recibe tráfico. Por eso en local va `kind`/`baremetal`
> o el addon de minikube, no `cloud`.

---

## 9. ConfigMaps y Secrets

```yaml
# configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.json: |
    {
      "port": 8080,
      "debug": false
    }
```

```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
stringData: # stringData = texto plano (K8s lo codifica en base64)
  username: admin
  password: mi-password-seguro
```

```yaml
# Usarlos en un Deployment
spec:
  containers:
    - name: app
      # Como variables de entorno
      envFrom:
        - configMapRef:
            name: app-config
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
      # Como archivos montados
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

```bash
# Crear desde CLI
kubectl create configmap app-config --from-literal=APP_ENV=production
kubectl create secret generic db-secrets --from-literal=password=mi-password
kubectl create configmap app-config --from-file=config.json
```

---

## 10. Persistent Volumes

```yaml
# pvc.yml (PersistentVolumeClaim)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```yaml
# Usar en un pod
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data
```

---

## 11. Namespaces (Organización)

```bash
# Crear
kubectl create namespace staging
kubectl create namespace production

# Aplicar en un namespace
kubectl apply -f deployment.yml -n staging

# Cambiar namespace por defecto
kubectl config set-context --current --namespace=staging

# Resource quotas por namespace
```

```yaml
# resource-quota.yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

---

## 12. Debugging

```bash
# Ver eventos del cluster
kubectl get events --sort-by=.metadata.creationTimestamp

# ¿Por qué un pod no arranca?
kubectl describe pod mi-pod          # Buscar en "Events"
kubectl logs mi-pod --previous       # Logs del contenedor anterior (si crasheó)

# Estado del pod
kubectl get pods
# STATUS comunes:
#   Running        = Todo bien
#   Pending        = Esperando (sin recursos o sin nodo)
#   CrashLoopBackOff = Crashea y reinicia en loop
#   ImagePullBackOff = No puede bajar la imagen
#   Error          = Falló
#   Completed      = Terminó (para Jobs)

# Entrar al pod
kubectl exec -it mi-pod -- /bin/sh

# Ver recursos del cluster
kubectl top nodes
kubectl top pods

# Verificar DNS
kubectl exec -it mi-pod -- nslookup mi-servicio.default.svc.cluster.local
```

---

## 13. Ejemplo Completo: App + DB

```yaml
# k8s/namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: mi-app
---
# k8s/postgres.yml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: mi-app
stringData:
  POSTGRES_PASSWORD: password-seguro
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: mi-app
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mi-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: mi-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
---
# k8s/app.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: mi-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: miusuario/mi-api:v1.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              value: "postgres://postgres:password-seguro@postgres:5432/app?sslmode=disable"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: mi-app
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: mi-app
spec:
  ingressClassName: nginx
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

```bash
# Aplicar todo
kubectl apply -f k8s/
```
