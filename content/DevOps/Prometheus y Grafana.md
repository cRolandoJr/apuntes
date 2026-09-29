[[0. Índice DevOps]]

# Prometheus y Grafana — Monitoreo y Observabilidad

## 1. ¿Qué son?

```
Prometheus = Recolector de métricas (números: CPU, RAM, requests/segundo, latencia)
Grafana    = Dashboard visual (gráficos bonitos de esas métricas)
Alertmanager = Enviar alertas (email, Slack, PagerDuty)

Flujo:
  App expone métricas → Prometheus las recolecta → Grafana las muestra → Alertmanager avisa
```

---

## 2. Prometheus

### Instalación (Docker Compose)

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123

volumes:
  prometheus_data:
  grafana_data:
```

### Configuración de Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s # Cada cuánto recolectar métricas
  evaluation_interval: 15s # Cada cuánto evaluar reglas/alertas

# Qué scrapear (targets)
scrape_configs:
  # Prometheus se monitorea a sí mismo
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Métricas del servidor (node-exporter)
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
        labels:
          instance: "web-server-1"

  # Tu aplicación
  - job_name: "mi-app"
    static_configs:
      - targets: ["mi-app:8080"]
    metrics_path: /metrics
    scrape_interval: 10s

  # Múltiples servidores
  - job_name: "servidores"
    static_configs:
      - targets:
          - "192.168.1.10:9100"
          - "192.168.1.11:9100"
          - "192.168.1.12:9100"
```

### En Kubernetes (con Helm)

```bash
# Instalar Prometheus + Grafana + AlertManager + Node Exporter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f monitoring-values.yml
```

```yaml
# monitoring-values.yml
grafana:
  adminPassword: mi-password-seguro
  ingress:
    enabled: true
    hosts:
      - grafana.ejemplo.com

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
```

---

## 3. PromQL — Consultas

```promql
# Métrica directa
node_cpu_seconds_total

# Filtrar por label
node_cpu_seconds_total{mode="idle"}
node_cpu_seconds_total{instance="web-server-1", mode!="idle"}

# Rate (tasa de cambio por segundo) — para contadores
rate(http_requests_total[5m])       # Requests/segundo en los últimos 5 min
rate(node_cpu_seconds_total{mode="idle"}[5m])

# Porcentaje de CPU usado
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memoria usada (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disco usado (%)
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Latencia (histograma)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))   # p95
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))   # p99

# Operaciones de agregación
sum(rate(http_requests_total[5m]))                    # Total requests/s
sum by(status_code)(rate(http_requests_total[5m]))    # Por código HTTP
avg by(instance)(node_load1)                          # Carga promedio por instancia
max(node_memory_MemTotal_bytes)                       # Máximo
count(up == 1)                                        # Cantidad de targets up

# Tasa de errores
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

---

## 4. Alertas

```yaml
# alert-rules.yml (cargar en Prometheus)
groups:
  - name: servidores
    rules:
      - alert: ServidorCaido
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Servidor {{ $labels.instance }} está caído"
          description: "{{ $labels.instance }} no responde hace más de 2 minutos."

      - alert: CPUAlta
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU alta en {{ $labels.instance }}"
          description: "CPU por encima del 85% hace 5 minutos. Actual: {{ $value }}%"

      - alert: DiscoLleno
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disco casi lleno en {{ $labels.instance }}"

      - alert: MemoriaAlta
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: critical

  - name: aplicacion
    rules:
      - alert: TasaErroresAlta
        expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Tasa de errores 5xx superior al 5%"

      - alert: LatenciaAlta
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latencia p95 superior a 1 segundo"
```

### AlertManager (enviar notificaciones)

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: "slack"
  group_by: ["alertname", "severity"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: "slack-critical"

receivers:
  - name: "slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/xxx/yyy/zzz"
        channel: "#alertas"
        title: "{{ .GroupLabels.alertname }}"
        text: "{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}"

  - name: "slack-critical"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/xxx/yyy/zzz"
        channel: "#alertas-criticas"
```

---

## 5. Grafana

### Configurar Datasource

1. Ir a Grafana (http://localhost:3000)
2. Configuration → Data Sources → Add data source
3. Seleccionar Prometheus
4. URL: `http://prometheus:9090`
5. Save & Test

### Dashboards Recomendados

Importar desde grafana.com/grafana/dashboards:

| Dashboard          | ID   | Uso                   |
| ------------------ | ---- | --------------------- |
| Node Exporter Full | 1860 | Métricas del servidor |
| Docker Dashboard   | 893  | Contenedores          |
| Kubernetes Cluster | 6417 | Cluster K8s           |
| Nginx              | 9614 | Nginx                 |
| PostgreSQL         | 9628 | PostgreSQL            |

```
Grafana → + → Import → Pegar ID → Load → Seleccionar datasource → Import
```

### Crear Dashboard Manual

1. **+ → New Dashboard → Add visualization**
2. Seleccionar datasource (Prometheus)
3. Escribir query PromQL
4. Elegir tipo de gráfico (Time series, Gauge, Stat, Table)
5. Configurar thresholds (colores por valor)
6. Save dashboard

---

## 6. Instrumentar tu App (Exponer Métricas)

### Go

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    httpRequests = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total de requests HTTP",
        },
        []string{"method", "path", "status_code"},
    )

    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Duración de requests HTTP",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequests, httpDuration)
}

func main() {
    // Endpoint de métricas
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

---

## 7. Los 4 Golden Signals (Google SRE)

Qué monitorear en cualquier servicio:

| Signal         | Qué mide                             | PromQL Ejemplo                                                             |
| -------------- | ------------------------------------ | -------------------------------------------------------------------------- |
| **Latency**    | Tiempo de respuesta                  | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` |
| **Traffic**    | Requests por segundo                 | `sum(rate(http_requests_total[5m]))`                                       |
| **Errors**     | Tasa de errores                      | `sum(rate(http_requests_total{code=~"5.."}[5m]))`                          |
| **Saturation** | Qué tan lleno está (CPU, RAM, disco) | `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100`           |
