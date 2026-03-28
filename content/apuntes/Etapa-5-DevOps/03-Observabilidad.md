# Observabilidad en Go

Observabilidad = entender qué está pasando en producción sin tener acceso directo al sistema. Las tres bases son: **logs**, **métricas** y **trazas**.

---

## Logging estructurado con slog

`slog` es el logger estándar de Go desde la versión 1.21. Produce logs estructurados en JSON — esencial para sistemas de análisis de logs (Loki, CloudWatch, Datadog).

```go
import "log/slog"

// Setup en main.go
func setupLogger(env string) *slog.Logger {
    var handler slog.Handler

    if env == "production" {
        // JSON — legible para máquinas (Loki, CloudWatch, etc.)
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level:     slog.LevelInfo,
            AddSource: true, // añade file:line al log
        })
    } else {
        // Texto — legible para humanos en desarrollo
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelDebug,
        })
    }

    return slog.New(handler)
}

// Uso básico
logger.Info("servidor iniciado", "port", 8080, "env", "production")
logger.Error("error al crear producto", "error", err, "name", input.Name)
logger.With("requestId", id).Debug("procesando request")
```

### Logger contextual — propagar campos comunes

```go
// Middleware — añadir requestId, userId al contexto para todos los logs del request
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            requestID := uuid.New().String()

            // Logger enriquecido para este request
            reqLogger := logger.With(
                "requestId", requestID,
                "method", r.Method,
                "path", r.URL.Path,
                "ip", r.RemoteAddr,
            )

            // Inyectar en contexto para que los usecases también lo usen
            ctx := WithLogger(r.Context(), reqLogger)

            // Wrappear ResponseWriter para capturar status code
            wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

            next.ServeHTTP(wrapped, r.WithContext(ctx))

            reqLogger.Info("request completado",
                "status", wrapped.statusCode,
                "duration_ms", time.Since(start).Milliseconds(),
            )
        })
    }
}

// Key para el logger en context
type loggerKey struct{}

func WithLogger(ctx context.Context, l *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey{}, l)
}

func FromContext(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}

// En el usecase — usar el logger del contexto
func (uc *productUseCase) Create(ctx context.Context, input domain.CreateProductInput) (*domain.Product, error) {
    log := logger.FromContext(ctx)
    log.Info("creando producto", "name", input.Name, "price", input.Price)

    product, err := uc.repo.Create(ctx, &domain.Product{...})
    if err != nil {
        log.Error("error al crear producto en repo", "error", err)
        return nil, err
    }

    log.Info("producto creado", "id", product.ID)
    return product, nil
}
```

---

## Métricas con Prometheus

Prometheus es el estándar de facto para métricas en backend. Hace scraping del endpoint `/metrics` de tu app.

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

### Definir métricas

```go
// internal/infra/metrics/metrics.go

package metrics

import "github.com/prometheus/client_golang/prometheus"

// Los 4 tipos de métricas:
// Counter    → solo sube (requests totales, errores totales)
// Gauge      → sube y baja (conexiones activas, tamaño de cola)
// Histogram  → distribución (latencia, tamaño de request)
// Summary    → percentiles precomputados (menos flexible que histograma)

var (
    // Counter — total de requests HTTP
    HTTPRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total de requests HTTP por método, ruta y status",
        },
        []string{"method", "path", "status"},
    )

    // Histogram — latencia de requests HTTP
    HTTPRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Duración de requests HTTP en segundos",
            Buckets: prometheus.DefBuckets, // .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
        },
        []string{"method", "path"},
    )

    // Gauge — productos activos en la base de datos
    ProductsTotal = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "products_total",
        Help: "Total de productos activos en el sistema",
    })

    // Counter — operaciones de base de datos
    DBOperationsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "db_operations_total",
            Help: "Total de operaciones de base de datos",
        },
        []string{"operation", "table", "result"},
    )

    // Histogram — latencia de operaciones de DB
    DBOperationDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "db_operation_duration_seconds",
            Help:    "Duración de operaciones de base de datos",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
        },
        []string{"operation", "table"},
    )
)

func Register(reg prometheus.Registerer) {
    reg.MustRegister(
        HTTPRequestsTotal,
        HTTPRequestDuration,
        ProductsTotal,
        DBOperationsTotal,
        DBOperationDuration,
    )
}
```

### Middleware HTTP para métricas

```go
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        statusStr := strconv.Itoa(wrapped.statusCode)

        // Normalizar paths para evitar explosión de labels (cardinalidad alta)
        // /products/123 → /products/{id}
        path := chi.RouteContext(r.Context()).RoutePattern()
        if path == "" {
            path = r.URL.Path
        }

        metrics.HTTPRequestsTotal.WithLabelValues(r.Method, path, statusStr).Inc()
        metrics.HTTPRequestDuration.WithLabelValues(r.Method, path).Observe(duration)
    })
}
```

### Exponer el endpoint /metrics

```go
// cmd/main.go
import "github.com/prometheus/client_golang/prometheus/promhttp"

func main() {
    // Crear un registry personalizado (en lugar del global)
    reg := prometheus.NewRegistry()

    // Registrar métricas del sistema (Go runtime, memoria, GC)
    reg.MustRegister(collectors.NewGoCollector())
    reg.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))

    // Registrar métricas de la app
    metrics.Register(reg)

    // Router principal (la app)
    appRouter := setupAppRouter(...)

    // Router de métricas/salud (separado, posiblemente en otro puerto)
    metricsRouter := http.NewServeMux()
    metricsRouter.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
    metricsRouter.HandleFunc("/health", healthHandler)
    metricsRouter.HandleFunc("/ready", readyHandler)

    // Dos servidores: app en :8080, métricas/health en :9090
    go http.ListenAndServe(":9090", metricsRouter)
    http.ListenAndServe(":8080", appRouter)
}
```

---

## Health checks

```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    // Liveness — "¿el proceso está vivo?"
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

type ReadinessChecker struct {
    db   *pgxpool.Pool
    // otros recursos...
}

func (c *ReadinessChecker) readyHandler(w http.ResponseWriter, r *http.Request) {
    // Readiness — "¿listo para recibir tráfico?"
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    checks := map[string]string{}
    allOk := true

    if err := c.db.Ping(ctx); err != nil {
        checks["database"] = "unhealthy: " + err.Error()
        allOk = false
    } else {
        checks["database"] = "healthy"
    }

    status := http.StatusOK
    if !allOk {
        status = http.StatusServiceUnavailable
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "status": map[bool]string{true: "ready", false: "not ready"}[allOk],
        "checks": checks,
    })
}
```

---

## Los 4 Golden Signals (Google SRE)

Monitorear principalmente estas métricas:

| Signal         | Qué medir                               | Métrica de ejemplo                   |
| -------------- | --------------------------------------- | ------------------------------------ |
| **Latencia**   | Tiempo de respuesta (P99, P95, mediana) | `http_request_duration_seconds`      |
| **Tráfico**    | Requests por segundo                    | `http_requests_total` rate           |
| **Errores**    | Tasa de error (5xx / total)             | `http_requests_total{status=~"5.."}` |
| **Saturación** | ¿Cuánto le queda de capacidad?          | goroutines, conexiones DB, CPU       |

---

## Práctica: Novato vs Profesional

### Novato

```go
// fmt.Println en producción — no estructurado, difícil de analizar
fmt.Println("Error:", err)
fmt.Printf("Creating product: %s\n", name)

// Sin métricas — no sabés qué endpoints fallan ni cuánto tardan
```

### Profesional

```go
// Logs estructurados con contexto
log.Error("error al crear producto",
    "error", err,
    "productName", name,
    "requestId", requestID,  // correlacionar con traza
    "userId", userID,
)

// Métricas que alertan automáticamente
// Alerta: tasa de error > 1% por 5 minutos
// Alerta: P99 latencia > 500ms por 3 minutos
// Alerta: ProductsTotal drop > 20% (posible bug en delete)
```
