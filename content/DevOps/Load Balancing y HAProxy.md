[[0. Índice DevOps]]

# Load Balancing y HAProxy

## 1. ¿Qué es un Load Balancer?

Distribuye el tráfico entre múltiples servidores para:

- **Alta disponibilidad:** Si un server cae, el tráfico va a los otros
- **Escalabilidad:** Agregar más servers para manejar más carga
- **Mantenimiento:** Sacar un server sin downtime

```
                    ┌─── Server 1 (app)
Cliente → LB ──────┤─── Server 2 (app)
                    └─── Server 3 (app)
```

---

## 2. Tipos de Load Balancing

| Capa                  | Tipo                  | Qué balancea                           |
| --------------------- | --------------------- | -------------------------------------- |
| **Layer 4** (TCP/UDP) | Balanceo por conexión | No inspecciona contenido. Rápido       |
| **Layer 7** (HTTP)    | Balanceo por request  | Puede rutear por URL, headers, cookies |

---

## 3. Algoritmos de Balanceo

| Algoritmo    | Cómo funciona                                            |
| ------------ | -------------------------------------------------------- |
| `roundrobin` | Rotar entre servers en orden (A → B → C → A)             |
| `leastconn`  | Enviar al server con menos conexiones activas            |
| `source`     | Mismo cliente siempre va al mismo server (sticky por IP) |
| `uri`        | Mismo URI siempre va al mismo server (útil para cache)   |
| `random`     | Server aleatorio                                         |

---

## 4. Nginx como Load Balancer

Ya lo tenés en la vault de SysAdmin, pero acá está la config de load balancing:

```nginx
# /etc/nginx/conf.d/loadbalancer.conf

upstream backend {
    # Round robin (por defecto)
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;

    # Least connections
    # least_conn;

    # IP hash (sticky sessions)
    # ip_hash;

    # Server con peso (recibe el doble)
    # server 192.168.1.10:8080 weight=2;

    # Server de backup (solo si los otros caen)
    # server 192.168.1.99:8080 backup;

    # Health checks pasivos
    # server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name app.ejemplo.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

---

## 5. HAProxy

HAProxy es un load balancer dedicado. Más potente que nginx para balanceo puro.

### Instalación

```bash
sudo apt install haproxy
sudo systemctl enable --now haproxy
```

### Configuración Básica

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

    # SSL hardening
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    retries 3

# --- Frontend (lo que recibe el tráfico) ---
frontend web
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/mi-cert.pem

    # Redirect HTTP → HTTPS
    http-request redirect scheme https unless { ssl_fc }

    # Rutear por path
    acl is_api path_beg /api
    acl is_static path_beg /static

    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend app_servers

# --- Backends (los servidores) ---
backend app_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server app1 192.168.1.10:8080 check inter 5s fall 3 rise 2
    server app2 192.168.1.11:8080 check inter 5s fall 3 rise 2
    server app3 192.168.1.12:8080 check inter 5s fall 3 rise 2

backend api_servers
    balance leastconn
    option httpchk GET /api/health

    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check

backend static_servers
    balance roundrobin
    server static1 192.168.1.30:80 check

# --- Stats (dashboard) ---
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
```

### Comandos

```bash
# Verificar configuración
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Recargar sin downtime
sudo systemctl reload haproxy

# Ver stats en el browser
# http://tu-server:8404/stats

# Logs
sudo journalctl -u haproxy -f
```

---

## 6. Health Checks

```haproxy
# HTTP health check
backend app_servers
    option httpchk GET /health
    http-check expect status 200

    # check = habilitar health check
    # inter 5s = verificar cada 5 segundos
    # fall 3 = marcar como down después de 3 fallos
    # rise 2 = marcar como up después de 2 éxitos
    server app1 192.168.1.10:8080 check inter 5s fall 3 rise 2

# TCP health check (Layer 4)
backend db_servers
    mode tcp
    option tcp-check
    server db1 192.168.1.50:5432 check
```

---

## 7. Sticky Sessions

```haproxy
# Por cookie (recomendado)
backend app_servers
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server app1 192.168.1.10:8080 check cookie s1
    server app2 192.168.1.11:8080 check cookie s2

# Por IP de origen
backend app_servers
    balance source
    hash-type consistent
    server app1 192.168.1.10:8080 check
    server app2 192.168.1.11:8080 check
```

---

## 8. Rate Limiting

```haproxy
frontend web
    bind *:80

    # Tabla para tracking de requests por IP
    stick-table type ip size 100k expire 30s store http_req_rate(10s)

    # Trackear
    http-request track-sc0 src

    # Denegar si más de 100 requests en 10 segundos
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
```

---

## 9. Layer 4 (TCP) — Balanceo de Base de Datos

```haproxy
frontend postgres
    mode tcp
    bind *:5432
    default_backend postgres_servers

backend postgres_servers
    mode tcp
    balance roundrobin
    option tcp-check

    server pg1 192.168.1.50:5432 check
    server pg2 192.168.1.51:5432 check
    server pg3 192.168.1.52:5432 check backup
```

---

## 10. HAProxy vs Nginx vs Cloud LB

| Característica  | HAProxy       | Nginx              | Cloud LB (ALB) |
| --------------- | ------------- | ------------------ | -------------- |
| Performance L4  | Excelente     | Buena              | Buena          |
| Performance L7  | Excelente     | Excelente          | Buena          |
| Stats/Dashboard | Incluido      | Requiere Plus      | CloudWatch     |
| Config          | Archivo único | Múltiples archivos | API/Console    |
| SSL Termination | Sí            | Sí                 | Sí             |
| WebSockets      | Sí            | Sí                 | Sí             |
| Costo           | Gratis        | Gratis             | Pago por uso   |
| Caso de uso     | LB dedicado   | LB + Web server    | Cloud nativo   |

> **Recomendación:** En cloud usar el LB del provider (ALB/NLB). On-premise o hybrid, HAProxy o nginx.
