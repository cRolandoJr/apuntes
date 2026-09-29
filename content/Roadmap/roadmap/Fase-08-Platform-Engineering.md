# Fase 8: Platform Engineering Senior — Meses 22 a 24

> **Objetivo**: Integrar todo lo aprendido en las 7 fases anteriores para diseñar, construir y operar **plataformas internas** que permiten a los equipos de desarrollo deployar y operar sus aplicaciones de forma autónoma. Este es el nivel senior: ya no configurás cosas, diseñás sistemas.

---

## Mes 22: GitOps y Delivery Avanzado

### 22.1 ¿Qué es GitOps?

GitOps es el principio de que **Git es la única fuente de verdad** para la infraestructura y las aplicaciones. No se aplican cambios manualmente ni con `kubectl apply` — todo pasa por Git.

```
ANTES (imperativo):
  Developer → kubectl apply → Cluster
  Problema: ¿Quién cambió qué? ¿Cuándo? No hay auditoría.

GITOPS (declarativo):
  Developer → Git push → ArgoCD detecta cambio → Aplica al cluster
  Todo es trazable, revertible, revisable.
```

**ArgoCD**: El controller de GitOps más popular para Kubernetes.

```yaml
# ArgoCD Application: "Vigilá este repo y aplicá cambios al cluster"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gestion-productos
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rolando/gestion-productos.git
    targetRevision: main
    path: k8s/ # Carpeta con los manifiestos YAML
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true # Borrar recursos que ya no existen en Git
      selfHeal: true # Si alguien cambia algo manualmente, revertir a Git
    syncOptions:
      - CreateNamespace=true
```

**Flujo GitOps completo**:

```
1. Developer hace PR con cambios en k8s/deployment.yaml
2. CI pipeline: tests, lint, build image, push image
3. PR aprobado y mergeado a main
4. ArgoCD detecta el nuevo commit
5. ArgoCD compara Git (deseado) vs Cluster (actual)
6. ArgoCD aplica las diferencias al cluster
7. Si algo falla → ArgoCD muestra "OutOfSync" y alerta
8. Para rollback: git revert → ArgoCD aplica el revert
```

### 22.2 Estrategias de Deploy Avanzadas

**Rolling Update** (ya lo conocés):

```
v1 v1 v1 v1    → Gradualmente reemplaza
v2 v1 v1 v1    → Uno nuevo, uno viejo
v2 v2 v1 v1    → ...
v2 v2 v2 v1    → ...
v2 v2 v2 v2    → Listo
```

**Blue-Green**:

```
Blue (v1): ████████  ← Tráfico va aquí
Green (v2): ████████  ← Está listo pero sin tráfico

Switch:
Blue (v1): ████████  ← Sin tráfico (standby para rollback)
Green (v2): ████████  ← TODO el tráfico ahora va aquí

Rollback instantáneo: cambiar el switch de vuelta a Blue
```

**Canary** (el más sofisticado):

```
v1: ████████████████████  95% del tráfico
v2: █                      5% del tráfico

Si v2 está sano (métricas ok):
v1: ██████████████████    90% del tráfico
v2: ██                    10% del tráfico

Si v2 sigue sano:
v1: ██████████            50%
v2: ██████████            50%

Finalmente:
v1: (terminado)
v2: ████████████████████  100%

Si en CUALQUIER paso las métricas se degradan → rollback automático
```

Herramientas para canary: **Argo Rollouts**, **Flagger**, **Istio**.

### 22.3 Feature Flags

En vez de deployar código nuevo a todos, controlás quién ve qué feature:

```go
// En vez de:
func getProducts() {
    // Código nuevo arriesgado
}

// Con feature flags:
func getProducts() {
    if featureFlags.IsEnabled("new-product-search", user) {
        return newProductSearch()  // Solo para usuarios seleccionados
    }
    return oldProductSearch()     // El resto usa la versión vieja
}
```

Herramientas: **LaunchDarkly**, **Unleash** (open source), **Flagsmith** (open source).

---

## Mes 23: Platform Engineering

### 23.1 ¿Qué es una Internal Developer Platform (IDP)?

Es el conjunto de herramientas, workflows y servicios que tu equipo de Platform construye para que los developers puedan **deployar, monitorear y operar sus aplicaciones sin depender de ops**.

```
┌──── Lo que un developer necesita ────┐
│                                       │
│ "Quiero deployar mi servicio"         │
│ "Quiero una base de datos"            │
│ "Quiero ver los logs"                 │
│ "Quiero saber si mi servicio anda"    │
│ "Quiero hacer rollback"              │
│                                       │
└───────────────┬───────────────────────┘
                │
    ┌───────────┴──── IDP ──────────────┐
    │                                    │
    │  ┌──────────┐  ┌──────────────┐  │
    │  │ Templates │  │ Self-service │  │
    │  │ (Helm,    │  │ Portal       │  │
    │  │  Kustomize)│  │ (Backstage)  │  │
    │  └──────────┘  └──────────────┘  │
    │  ┌──────────┐  ┌──────────────┐  │
    │  │ CI/CD    │  │ Observability│  │
    │  │ Pipelines│  │ Stack        │  │
    │  └──────────┘  └──────────────┘  │
    │  ┌──────────┐  ┌──────────────┐  │
    │  │ GitOps   │  │ Secret       │  │
    │  │ (ArgoCD) │  │ Management   │  │
    │  └──────────┘  └──────────────┘  │
    │                                    │
    └────────────────────────────────────┘
                │
    ┌───────────┴───────────────────────┐
    │         Kubernetes Cluster         │
    │  ┌─────┐ ┌─────┐ ┌─────┐        │
    │  │Svc A│ │Svc B│ │Svc C│        │
    │  └─────┘ └─────┘ └─────┘        │
    └────────────────────────────────────┘
```

### 23.2 Los Componentes de una IDP

**1. Service Templates**: Plantillas para crear nuevos servicios.

```yaml
# template.yaml — Backstage template
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: go-service
  title: Go Microservice
  description: Crea un nuevo microservicio en Go con CI/CD y observabilidad
spec:
  parameters:
    - title: Información del servicio
      properties:
        name:
          title: Nombre del servicio
          type: string
        owner:
          title: Equipo owner
          type: string
        port:
          title: Puerto
          type: number
          default: 8080
  steps:
    - id: create-repo
      action: github:repo:create
    - id: scaffold
      action: fetch:template
      input:
        url: ./skeleton # Template con Dockerfile, Helm chart, CI pipeline, etc.
    - id: register
      action: catalog:register
```

Un developer llena el formulario → se crea un repo con TODO listo: código base, Dockerfile, Helm chart, pipeline CI/CD, dashboards de Grafana, alertas de Prometheus. **Zero to production en minutos.**

**2. Backstage**: Portal de desarrolladores de Spotify (open source).

- Catálogo de servicios (quién es el owner de cada servicio)
- Templates para crear servicios nuevos
- Documentación centralizada
- Integración con CI/CD, monitoreo, incidentes

**3. Crossplane**: Crear recursos de cloud desde Kubernetes (la BD, el bucket S3, etc.).

```yaml
# "Quiero una base de datos PostgreSQL en AWS"
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: mi-database
spec:
  forProvider:
    engine: postgres
    engineVersion: "16"
    instanceClass: db.t3.micro
    masterUsername: admin
    allocatedStorage: 20
    region: us-east-1
```

El developer pide la BD con un YAML en Git → Crossplane la crea en AWS automáticamente.

### 23.3 Service Mesh (Istio/Linkerd)

Un Service Mesh gestiona la comunicación entre servicios de forma transparente:

```
SIN Mesh:
Service A → (HTTP directo) → Service B
Problemas: ¿Retry? ¿Timeout? ¿Circuit breaker? ¿mTLS? Cada servicio lo implementa.

CON Mesh:
Service A → [Sidecar Proxy] → [Sidecar Proxy] → Service B
El proxy maneja: retries, timeouts, circuit breakers, mTLS, observabilidad.
Los servicios no se enteran. Es transparente.
```

**¿Qué resuelve?**

- **mTLS automático**: Toda la comunicación entre servicios es encriptada
- **Retries y timeouts**: Configurables por servicio, sin cambiar código
- **Circuit breaker**: Si un servicio falla mucho, dejar de llamarlo
- **Traffic splitting**: Enviar 5% del tráfico a la versión canary
- **Observabilidad**: Métricas y traces automáticos de toda la comunicación

### 23.4 DevSecOps — Seguridad Integrada

La seguridad no es una fase al final — está en cada paso:

```
Code → SAST (análisis estático de código)
Build → SCA (verificar dependencias con vulnerabilidades conocidas)
Container → Scan de imagen (Trivy, Snyk)
Deploy → Admission controllers (prohibir containers root)
Runtime → Falco (detectar comportamiento anómalo)
```

```yaml
# Kyverno: Policy engine para K8s
# "Prohibir containers como root"
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce # Rechazar, no solo alertar
  rules:
    - name: check-non-root
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Los containers deben correr como non-root"
        pattern:
          spec:
            containers:
              - securityContext:
                  runAsNonRoot: true
```

---

## Mes 24: Integración Final y Nivel Senior

### 24.1 ¿Qué Hace un Senior Platform Engineer?

No solo configura herramientas. **Diseña sistemas** y toma decisiones con impacto organizacional:

1. **Diseñar la plataforma**: Qué herramientas, qué workflows, qué abstracciones
2. **Definir estándares**: Cómo se deployea, cómo se monitorea, cómo se documenta
3. **Automatizar todo lo repetitivo**: Si lo hacés 2 veces manualmente, automatizalo
4. **Evangelizar**: Convencer y enseñar a los equipos a usar la plataforma
5. **Medir el impacto**: "Desde que implementamos X, el tiempo de deploy bajó de 2 horas a 5 minutos"
6. **Gestionar incidentes complejos**: Ser el escalation point cuando todo falla
7. **Decisiones de arquitectura**: Documentar ADRs (Architecture Decision Records)

### 24.2 Architecture Decision Records (ADRs)

```markdown
# ADR-001: Usar ArgoCD para GitOps

## Status

Aceptado

## Contexto

Necesitamos un mecanismo declarativo para deployar a Kubernetes.
Los deploys manuales con kubectl causan inconsistencias y no tienen auditoría.

## Decisión

Usaremos ArgoCD como controlador de GitOps.

## Alternativas consideradas

- **FluxCD**: Más liviano pero menos UI, menos adopción en el equipo
- **Jenkins CD**: Imperativo, no GitOps nativo
- **Spinnaker**: Demasiado complejo para nuestro tamaño

## Consecuencias

- ✅ Todo cambio pasa por Git (auditable, reversible)
- ✅ Reconciliation automático (self-heal)
- ⚠️ Requiere formación del equipo en GitOps
- ⚠️ Dependencia nueva (ArgoCD como servicio crítico)
```

### 24.3 Skills Blandas del Senior

- **Escribir RFCs/propuestas**: Antes de construir, documentar qué y por qué
- **Dar feedback en code reviews**: No solo "esto está mal", sino "consideraste hacer X porque Y"
- **Mentorear juniors**: Tu conocimiento no vale si muere con vos
- **Negociar con management**: "Necesitamos 2 sprints para mejorar la plataforma porque el tiempo de deploy está causando X"
- **Postmortems blameless**: La cultura de no culpar personas sino mejorar sistemas

---

## Proyecto Final: La Plataforma Completa

### "Internal Developer Platform MVP"

Construí una plataforma mínima pero funcional que integre TODO:

**Infraestructura (Terraform)**:

- Cluster Kubernetes (k3s o EKS)
- Networking (VPC, subnets)
- Base de datos (RDS o PostgreSQL en K8s)
- Bucket S3 para artefactos y backups

**Configuración (Ansible)**:

- Provisioning de nodos
- Instalación de herramientas base

**Platform Core (Kubernetes)**:

- ArgoCD para GitOps
- Prometheus + Grafana + Loki para observabilidad
- Cert-manager para TLS automático
- Ingress controller (nginx o traefik)
- Kyverno para policies de seguridad

**Developer Experience**:

- Template de servicio Go con:
  - Dockerfile optimizado
  - Helm chart
  - Pipeline CI (GitHub Actions)
  - Dashboard de Grafana pre-configurado
  - Alertas de Prometheus
- README con instrucciones de uso
- Runbooks para incidentes comunes

**Workflow demostrable**:

```bash
# 1. Un developer "crea un nuevo servicio" usando el template
# 2. Pushea código → CI compila, testea, escanea, pushea imagen
# 3. ArgoCD detecta el cambio y deploya al cluster
# 4. El servicio aparece en los dashboards de Grafana automáticamente
# 5. Si las métricas se degradan → alerta en Slack
# 6. Para rollback → git revert → ArgoCD lo aplica

# Verificación:
kubectl get applications -n argocd                  # ArgoCD tiene la app
kubectl get pods -n production                      # Pods running
curl https://api.tudominio.com/query                # API responde con TLS
# Grafana muestra métricas automáticamente
# Loki muestra logs
# ArgoCD muestra estado "Synced"
```

---

## Recursos Finales

### Platform Engineering

1. **platformengineering.org** — Comunidad y recursos
2. **"Team Topologies" de Skelton & Pais** — Cómo organizar equipos de platform
3. **"Platform Engineering on Kubernetes" de Mauricio Salatino** — Práctico
4. **Backstage** (backstage.io) — Portal de desarrolladores open source
5. **CNCF Landscape** (landscape.cncf.io) — Mapa de todas las herramientas del ecosistema

### GitOps

1. **"GitOps and Kubernetes" de Billy Yuen et al.** — Comprensivo
2. **ArgoCD Docs** (argo-cd.readthedocs.io) — Bien documentado

### Nivel Senior

1. **"The Staff Engineer's Path" de Tanya Reilly** — Cómo ser un IC senior
2. **"An Elegant Puzzle" de Will Larson** — Gestión de equipos de engineering
3. **"Accelerate" de Forsgren, Humble & Kim** — Métricas que importan (DORA metrics)
4. **StaffEng.com** — Historias de staff engineers

### Certificaciones pendientes

- **CKS** (Certified Kubernetes Security Specialist) → Rendila al final
- **AWS SAA** (si no la rendiste en la Fase 6) → Ahora ya tenés experiencia

---

## Perfil Final al Mes 24

Al terminar las 8 fases, tu perfil incluye:

**Conocimiento**:

- Linux/Networking a nivel profundo
- Go + Python para automatización y herramientas
- Docker + Kubernetes (con CKA)
- Terraform + Ansible (IaC completo)
- AWS (o GCP/Azure)
- Prometheus + Grafana + Loki (observabilidad)
- ArgoCD + GitOps
- SRE practices (SLOs, postmortems, incident management)
- Platform Engineering (IDP, service templates)
- DevSecOps (scanning, policies)

**Certificaciones** (opcionales pero valiosas):

- LPIC-1
- CKA
- Terraform Associate
- AWS Solutions Architect Associate
- CKS

**Proyectos en tu portfolio**:

1. Script de monitoreo de sistema (Fase 1)
2. Servidor hardenizado con servicios systemd (Fase 2)
3. API REST en Go con PostgreSQL y tests (Fase 3)
4. Pipeline CI/CD completo (Fase 4)
5. Stack completo en Kubernetes (Fase 5)
6. Infraestructura con Terraform + Ansible (Fase 6)
7. Stack de observabilidad (Fase 7)
8. Internal Developer Platform (Fase 8)

**Empleabilidad**: Con este perfil, podés aplicar a posiciones de:

- **Platform Engineer** (Mid/Senior)
- **DevOps Engineer** (Senior)
- **SRE** (Site Reliability Engineer)
- **Cloud Engineer** (Senior)
- **Infrastructure Engineer** (Senior)

Salarios de referencia (2026, remoto LATAM para empresas US/EU):

- Mid: $3,000 - $5,000 USD/mes
- Senior: $5,000 - $10,000 USD/mes
- Staff/Principal: $10,000+ USD/mes

---

## Mensaje Final

Rolando: este documento tiene todo lo que necesitás para llegar. No todo, pero sí el mapa completo y las herramientas para profundizar cada tema.

La clave no es leer todo esto de una sentada — es seguir el plan fase por fase, ejercicio por ejercicio, sin saltear.

Cada vez que te frustres, recordá: hace 24 meses no sabías qué era un syscall. Ahora diseñás plataformas.

Los mejores Platform Engineers no son los que saben más herramientas. Son los que entienden los **fundamentos** tan bien que pueden aprender cualquier herramienta nueva en días.

Empezá por la Fase 1, Mes 1, ejercicio 1. Lo demás viene solo.
