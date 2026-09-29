[[0. Índice DevOps]]

# Logging Centralizado

## 1. ¿Por qué Centralizar Logs?

```
Problema: 10 servidores, cada uno con sus logs en /var/log/
  - ¿Dónde está el error? → SSH a cada server y revisar
  - Un server cae → perdés los logs

Solución: Todos los logs van a un lugar central, con búsqueda y visualización
```

### Stacks Populares

| Stack   | Componentes                       | Cuándo usarlo                       |
| ------- | --------------------------------- | ----------------------------------- |
| **ELK** | Elasticsearch + Logstash + Kibana | El clásico, potente pero pesado     |
| **EFK** | Elasticsearch + Fluentd + Kibana  | Kubernetes (Fluentd como DaemonSet) |
| **PLG** | Promtail + Loki + Grafana         | Liviano, ideal con Prometheus       |

---

## 2. Loki + Grafana (Recomendado)

Loki es el "Prometheus de los logs". No indexa el contenido del log (como Elasticsearch) sino solo los labels, lo que lo hace mucho más liviano.

### Instalación con Docker Compose

```yaml
# docker-compose.yml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro # Logs del host
      - /var/lib/docker/containers:/var/lib/docker/containers:ro # Logs de Docker
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123

volumes:
  loki_data:
  grafana_data:
```

### Configuración de Loki

```yaml
# loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d
```

### Configuración de Promtail

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Logs del sistema
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: web-server-1
          __path__: /var/log/syslog

  # Logs de nginx
  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          host: web-server-1
          __path__: /var/log/nginx/*.log

  # Logs de tu app
  - job_name: mi-app
    static_configs:
      - targets: [localhost]
        labels:
          job: mi-app
          host: web-server-1
          __path__: /var/log/mi-app/*.log

  # Logs de Docker
  - job_name: docker
    docker_sd_configs:
      - host: "unix:///var/run/docker.sock"
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ["__meta_docker_container_name"]
        target_label: "container"
```

### Consultar en Grafana

1. Agregar Data Source → Loki → URL: `http://loki:3100`
2. Ir a Explore → Seleccionar Loki

```logql
# LogQL — Lenguaje de consulta de Loki

# Todos los logs de un job
{job="nginx"}

# Filtrar por host
{job="syslog", host="web-server-1"}

# Buscar texto
{job="nginx"} |= "error"
{job="nginx"} |= "404"

# Regex
{job="nginx"} |~ "status=(4|5)[0-9]{2}"

# Excluir
{job="syslog"} != "CRON"

# Parsear campos (logs estructurados)
{job="nginx"} | json | status >= 500

# Contar errores por segundo
rate({job="nginx"} |= "error" [5m])

# Top 10 IPs con más errores
topk(10, sum by(remote_addr) (rate({job="nginx"} | json | status >= 500 [5m])))
```

---

## 3. En Kubernetes

```bash
# Instalar Loki stack con Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  -n monitoring --create-namespace \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

En K8s, Promtail corre como DaemonSet (uno por nodo) y recolecta stdout/stderr de todos los pods automáticamente.

---

## 4. ELK Stack (Alternativa Pesada)

```yaml
# docker-compose.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
  syslog {
    port => 5514
  }
}

filter {
  if [type] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

## 5. Logging Estructurado (Best Practice)

```
# MAL — Log no estructurado (difícil de parsear)
2026-03-25 14:30:00 ERROR: User login failed for john from 192.168.1.50

# BIEN — Log estructurado (JSON)
{"timestamp":"2026-03-25T14:30:00Z","level":"error","msg":"User login failed","user":"john","ip":"192.168.1.50","service":"auth"}
```

Ventajas del logging estructurado:

- Fácil de filtrar: `{job="auth"} | json | level="error" | user="john"`
- No necesitás regex para parsear
- Podés agregar campos sin romper el parseo

---

## 6. Buenas Prácticas

```
1. Usar logging estructurado (JSON) siempre que se pueda
2. Incluir: timestamp, level, message, service, request_id
3. Niveles de log: DEBUG < INFO < WARN < ERROR < FATAL
4. En producción: nivel INFO o WARN (no DEBUG)
5. No loggear datos sensibles (passwords, tokens, PII)
6. Configurar retención (30 días suele ser suficiente)
7. Alertar sobre patrones de error, no sobre logs individuales
8. Usar request_id/trace_id para seguir un request entre microservicios
```
