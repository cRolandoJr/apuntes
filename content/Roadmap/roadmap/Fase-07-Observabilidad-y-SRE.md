# Fase 7: Observabilidad & SRE — Meses 19 a 21

> **Objetivo**: Pasar de "¿está funcionando?" a "entiendo exactamente qué hace cada componente del sistema en todo momento". Métricas, logs, traces. SLIs, SLOs, error budgets. Gestión de incidentes. La diferencia entre un DevOps y un SRE es esta fase.

---

## Mes 19: Los 3 Pilares de la Observabilidad

### 19.1 ¿Observabilidad vs Monitoreo?

**Monitoreo**: "Las alertas me dicen QUE algo falló"
**Observabilidad**: "Los datos me permiten entender POR QUÉ falló, incluso si nunca anticipé ese fallo"

Los 3 pilares:

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ MÉTRICAS│  │  LOGS   │  │ TRACES  │
│ (Qué)   │  │ (Por qué)│  │ (Dónde) │
│         │  │         │  │         │
│Números  │  │Texto    │  │Camino de│
│en serie │  │del      │  │un       │
│temporal │  │evento   │  │request  │
└─────────┘  └─────────┘  └─────────┘
```

**Métricas**: Números que cambian en el tiempo → "CPU al 85%", "200 requests/segundo", "latencia p99 = 250ms"
**Logs**: Registros de eventos → "ERROR [2026-03-23 15:30:02] Failed to connect to database: timeout"
**Traces**: El camino de un request a través de múltiples servicios → "Request → API Gateway (2ms) → Auth Service (15ms) → Product Service (180ms) → PostgreSQL (150ms)"

### 19.2 Métricas con Prometheus + Grafana

**Prometheus**: Base de datos de series temporales. Recolecta métricas con un modelo **pull** (va a buscar métricas a tus servicios).

**Grafana**: Dashboards para visualizar las métricas.

```
Tu API ─expose→ /metrics ←scrape─ Prometheus ←query─ Grafana
                                     │
                                     └─alert→ Alertmanager → Slack/PagerDuty
```

**Instrumentar tu API Go con Prometheus**:

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Definir métricas
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total de requests HTTP",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Duración de requests HTTP",
            Buckets: []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5},
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration)
}

// Middleware que registra métricas
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        // Wrapper para capturar el status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        status := strconv.Itoa(wrapped.statusCode)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

// Exponer métricas en /metrics
http.Handle("/metrics", promhttp.Handler())
```

**Prometheus config** (`prometheus.yml`):

```yaml
global:
  scrape_interval: 15s # Cada 15 segundos recoge métricas

scrape_configs:
  - job_name: "api"
    static_configs:
      - targets: ["api:8081"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"] # Métricas del host (CPU, RAM, disco)

  - job_name: "postgres"
    static_configs:
      - targets: ["postgres-exporter:9187"] # Métricas de PostgreSQL
```

**PromQL**: Lenguaje de consulta de Prometheus.

```promql
# Requests por segundo en los últimos 5 minutos
rate(http_requests_total[5m])

# Requests con error (status 5xx)
rate(http_requests_total{status=~"5.."}[5m])

# Tasa de errores (% de requests con error)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Latencia percentil 99 (el 99% de requests tardan menos de esto)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Uso de CPU del host
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memoria disponible
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Espacio en disco
(node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

### 19.3 Logs con Loki + Grafana (o ELK)

**Stack Loki** (más liviano que ELK, se integra con Grafana):

```
Tu API ──stdout/stderr──→ Promtail ──push──→ Loki ←──query──→ Grafana
```

**Logging estructurado**: En vez de strings, logs en JSON.

```go
// MAL (texto plano):
log.Printf("Error al crear producto: %v", err)
// Sale: 2026/03/23 15:30:02 Error al crear producto: [INVALID_INPUT] precio negativo

// BIEN (estructurado con zerolog):
import "github.com/rs/zerolog/log"

log.Error().
    Err(err).
    Str("operation", "create_product").
    Str("product_name", input.Name).
    Float64("price", input.Price).
    Str("correlation_id", correlationID).
    Msg("Failed to create product")
// Sale: {"level":"error","operation":"create_product","product_name":"Laptop",
//        "price":-10,"correlation_id":"abc-123","error":"[INVALID_INPUT] precio negativo",
//        "message":"Failed to create product","time":"2026-03-23T15:30:02-03:00"}
```

¿Por qué JSON? Porque Loki/ELK pueden **indexar y filtrar** por cualquier campo:

- "Mostrame todos los errores de `create_product`"
- "Mostrame todos los requests con `correlation_id=abc-123`"
- "¿Cuántos errores de `INVALID_INPUT` hubo en la última hora?"

### 19.4 Alertas

```yaml
# Prometheus alerting rules
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m # Debe mantenerse 5 minutos antes de alertar (evitar falsos positivos)
        labels:
          severity: critical
        annotations:
          summary: "Tasa de errores > 5%"
          description: "La API tiene {{ $value | humanizePercentage }} de errores en los últimos 5 min"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latencia p99 > 1 segundo"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Disco al {{ $value | humanizePercentage }} disponible"
```

---

## Mes 20: SRE (Site Reliability Engineering)

### 20.1 ¿Qué es SRE?

SRE es la disciplina de aplicar **ingeniería de software a problemas de operaciones**. Google lo inventó.

Principio central: **100% de disponibilidad es imposible e innecesario**. Lo que importa es definir cuánta confiabilidad necesitás y trabajar dentro de ese marco.

### 20.2 SLI, SLO, SLA

```
SLI (Service Level Indicator) → Métrica que mide confiabilidad
    Ejemplo: "El 99.2% de requests responden en < 200ms"

SLO (Service Level Objective) → El objetivo que queremos cumplir
    Ejemplo: "El 99.5% de requests deben responder en < 200ms"

SLA (Service Level Agreement) → Contrato con el cliente
    Ejemplo: "Garantizamos 99.5% de requests en < 200ms o devolvemos plata"

          SLI           SLO           SLA
        (medición)    (objetivo)    (contrato)
          99.2%    <    99.5%    ≤   99.5%
           ↑              ↑            ↑
       Lo que tenemos  Lo que queremos  Lo que prometemos
```

### 20.3 Error Budget

Si tu SLO es 99.5%, tenés un **error budget** de 0.5% — eso es cuánta indisponibilidad podés "gastar".

En un mes de 30 días:

- 99.5% SLO → 0.5% error budget → **~2 horas 11 minutos** de downtime permitido
- 99.9% SLO → 0.1% error budget → **~43 minutos**
- 99.99% SLO → 0.01% error budget → **~4 minutos**

**¿Para qué sirve?**

- Si hay error budget sobrante → podemos hacer deploys más agresivos, experimentar
- Si el error budget se agota → freeze de deploys, enfocarse en estabilidad
- Es un **framework para tomar decisiones** entre velocidad y confiabilidad

### 20.4 Los 4 Golden Signals

Google define 4 métricas fundamentales para cualquier servicio:

1. **Latencia**: Cuánto tarda en responder
   - Medir p50, p90, p95, p99 (no solo el promedio)
   - Separar latencia de requests exitosos vs fallidos

2. **Tráfico**: Cuánta demanda recibe
   - Requests por segundo
   - Queries a la BD por segundo

3. **Errores**: Qué proporción falla
   - HTTP 5xx / total
   - Errores de negocio (AppError)

4. **Saturación**: Cuán "lleno" está el sistema
   - % de CPU usado
   - % de memoria
   - Conexiones a la BD usadas vs pool total

```promql
# Dashboard "Golden Signals" en Grafana:

# Latencia p99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Tráfico (requests/segundo)
sum(rate(http_requests_total[5m]))

# Tasa de errores
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Saturación CPU
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

### 20.5 Gestión de Incidentes

Cuando algo falla en producción:

```
1. DETECTAR    → Alerta salta (Prometheus → Alertmanager → Slack/PagerDuty)
2. RESPONDER   → On-call reconoce la alerta en < 5 minutos
3. TRIAGEAR    → ¿Cuál es el impacto? ¿Qué porcentaje de usuarios afecta?
4. MITIGAR     → Acción inmediata para reducir impacto (rollback, escalar, feature flag)
5. RESOLVER    → Fix definitivo
6. POSTMORTEM  → ¿Qué pasó? ¿Cómo evitarlo? SIN culpar personas.
```

**Postmortem template**:

```markdown
# Postmortem: [Título del incidente]

**Fecha**: 2026-03-23
**Duración**: 45 minutos (15:30 - 16:15 UTC)
**Impacto**: 30% de requests fallaron con error 500
**Detectado por**: Alerta de HighErrorRate

## Timeline

- 15:28 — Deploy v2.3.1 va a producción
- 15:30 — Alertas de error rate > 5%
- 15:32 — On-call reconoce la alerta
- 15:35 — Identifica que el deploy cambió la query de productos
- 15:38 — Rollback a v2.3.0
- 15:40 — Error rate vuelve a niveles normales
- 16:15 — Confirmado estable, incidente cerrado

## Causa raíz

La migración SQL de v2.3.1 agregó una columna NOT NULL sin default, causando
que los INSERT fallaran para productos existentes que no tenían ese campo.

## Qué funcionó

- Las alertas se dispararon en < 2 minutos
- El rollback fue rápido gracias al pipeline de CD

## Qué no funcionó

- No había test de integración que validara las migraciones SQL
- El deploy no fue canary (fue 100% de tráfico de golpe)

## Action Items

- [ ] Agregar test de migración SQL en CI → @rolando, deadline 2026-03-30
- [ ] Implementar canary deployments → @team, deadline 2026-04-15
- [ ] Agregar runbook para "error rate alto" → @rolando, deadline 2026-03-25
```

---

## Mes 21: Stack de Observabilidad Completo + Proyecto

### 21.1 Stack con Docker Compose

```yaml
# monitoring/docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./promtail.yml:/etc/promtail/config.yml

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro

volumes:
  prometheus_data:
  grafana_data:
```

---

## Proyecto Integrador de Fase 7

### "Observabilidad Completa de tu API"

1. **Instrumentar** tu API Go con métricas de Prometheus
2. **Logging estructurado** con zerolog en formato JSON
3. **Stack de monitoreo**: Prometheus + Grafana + Loki + Alertmanager
4. **Dashboard de Golden Signals** en Grafana
5. **Alertas** para: error rate > 5%, latencia p99 > 1s, disco < 10%
6. **Definir SLOs** para tu API:
   - Disponibilidad: 99.5%
   - Latencia p99: < 500ms
7. **Simular un incidente** (ej: matar la BD) y escribir un postmortem
8. **Runbook**: Documento paso a paso para los escenarios de fallo más comunes

**Verificación**:

```bash
# Stack levantado
curl http://localhost:9090/api/v1/targets  # Prometheus ve tus targets
curl http://localhost:3000/                 # Grafana funciona

# Métricas de tu API
curl http://localhost:8081/metrics | grep http_requests_total

# Generar tráfico
for i in $(seq 1 100); do
  curl -s http://localhost:8081/query -X POST \
    -H "Content-Type: application/json" \
    -d '{"query": "{ products { id name } }"}' > /dev/null
done

# Verificar en Grafana que las métricas aparecen
# Verificar que los logs en Loki muestran los requests
```

---

## Recursos para esta Fase

### Observabilidad

1. **"Observability Engineering" de Charity Majors et al.** — El libro moderno de observabilidad
2. **"Site Reliability Engineering" de Google** (sre.google/sre-book) — Gratis online. LA biblia de SRE.
3. **"The Site Reliability Workbook" de Google** (sre.google/workbook) — Más práctico que el anterior
4. **Prometheus Docs** (prometheus.io/docs) — Excelente documentación
5. **Grafana Tutorials** (grafana.com/tutorials) — Labs gratuitos

### SRE

1. **SRE Book** (sre.google/sre-book) — Gratis online
2. **"Implementing SLOs" de Alex Hidalgo** — Práctico, enfocado en SLIs/SLOs
3. **Google SRE Course** (sre.google/classroom) — Curso gratuito

### Práctica

- **KillerCoda** — Labs de Prometheus y Grafana
- **Prometheus Playground** (demo.promlabs.com) — PromQL en el navegador

---

## Checkpoint: ¿Estoy listo para la Fase 8?

- [ ] ¿Puedo instrumentar una app Go con métricas de Prometheus?
- [ ] ¿Puedo escribir queries PromQL para los 4 golden signals?
- [ ] ¿Puedo crear un dashboard en Grafana?
- [ ] ¿Puedo explicar la diferencia entre logs, métricas y traces?
- [ ] ¿Puedo configurar alertas en Prometheus?
- [ ] ¿Puedo explicar SLI, SLO, SLA y error budget?
- [ ] ¿Puedo escribir un postmortem?
- [ ] ¿Puedo implementar logging estructurado?

Si respondiste 6+ de 8: avanzá a la Fase 8.
