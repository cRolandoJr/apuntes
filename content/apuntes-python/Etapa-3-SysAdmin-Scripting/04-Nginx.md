# Nginx como Reverse Proxy

Nginx es el servidor web que va por delante de tu app (FastAPI, Go, Node). Recibe las conexiones HTTP/S del mundo, las valida, y las pasa hacia tu proceso interno. Tu app nunca se expone directamente a internet.

```
internet → Nginx (puerto 80/443) → uvicorn/gunicorn (127.0.0.1:8000)
```

---

## Instalación

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install nginx -y

# Verificar
nginx -v
sudo systemctl status nginx
```

---

## Estructura de archivos

```
/etc/nginx/
├── nginx.conf              # config global (no tocar casi nunca)
├── sites-available/        # configs disponibles (un archivo por app)
│   └── mi-app
└── sites-enabled/          # symlinks a sites-available (activados)
    └── mi-app -> ../sites-available/mi-app
```

Flujo de trabajo:

1. Crear el archivo en `sites-available/`
2. Activarlo con symlink en `sites-enabled/`
3. Recargar nginx

---

## Config básica (HTTP)

```nginx
# /etc/nginx/sites-available/mi-app
server {
    listen 80;
    server_name mi-dominio.com www.mi-dominio.com;

    # Pasar requests a la app
    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_http_version 1.1;

        # Headers importantes — sin estos la app no sabe la IP real del cliente
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSockets (si la app los usa)
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
# Activar
sudo ln -s /etc/nginx/sites-available/mi-app /etc/nginx/sites-enabled/

# Verificar sintaxis antes de recargar
sudo nginx -t

# Aplicar
sudo systemctl reload nginx
```

---

## SSL con Let's Encrypt (HTTPS)

```bash
sudo apt install certbot python3-certbot-nginx -y

# Obtener certificado y configurar nginx automáticamente
sudo certbot --nginx -d mi-dominio.com -d www.mi-dominio.com

# Certbot modifica el archivo de nginx y agrega auto-renovación al cron
# Verificar auto-renovación
sudo certbot renew --dry-run
```

Resultado — certbot deja la config así:

```nginx
server {
    listen 80;
    server_name mi-dominio.com www.mi-dominio.com;

    # Redirigir HTTP → HTTPS siempre
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name mi-dominio.com www.mi-dominio.com;

    ssl_certificate     /etc/letsencrypt/live/mi-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mi-dominio.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Múltiples apps en el mismo servidor (upstream)

```nginx
# Definir upstreams — permite load balancing y nombres claros
upstream fastapi_app {
    server 127.0.0.1:8000;
    # Con múltiples instancias (balanceo round-robin):
    # server 127.0.0.1:8001;
    # server 127.0.0.1:8002;
    keepalive 32;   # mantener conexiones abiertas
}

upstream go_app {
    server 127.0.0.1:9000;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name api.mi-dominio.com;
    # ... ssl config ...

    location /api/productos/ {
        proxy_pass http://go_app;
        # ...headers...
    }

    location / {
        proxy_pass http://fastapi_app;
        # ...headers...
    }
}
```

---

## Archivos estáticos

```nginx
server {
    # ...

    # Servir archivos estáticos directamente desde nginx (sin pasar a la app)
    location /static/ {
        alias /var/www/mi-app/static/;
        expires 30d;                    # cacheo del browser
        add_header Cache-Control "public, immutable";
        access_log off;                 # no loguear requests de estáticos
    }

    # Media uploads
    location /media/ {
        alias /var/www/mi-app/media/;
        expires 7d;
        add_header Cache-Control "public";
    }

    location / {
        proxy_pass http://fastapi_app;
        # ...
    }
}
```

---

## Rate limiting

```nginx
# En el bloque http{} de nginx.conf (o en un archivo incluido)
# Definir la zona de rate limiting (solo se declara una vez)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

server {
    # ...

    location / {
        # Aplicar rate limit — máx 10 req/seg por IP, burst de 20
        limit_req zone=api_limit burst=20 nodelay;
        limit_req_status 429;

        proxy_pass http://fastapi_app;
        # ...
    }

    location /auth/login {
        # Rate limit más estricto para login
        limit_req zone=login_limit burst=5 nodelay;
        limit_req_status 429;

        proxy_pass http://fastapi_app;
        # ...
    }
}
```

---

## Timeouts y buffers (tuning básico)

```nginx
server {
    # ...

    # Timeouts
    proxy_connect_timeout 10s;    # tiempo para conectarse al upstream
    proxy_send_timeout    60s;    # tiempo para enviar request al upstream
    proxy_read_timeout    60s;    # tiempo para recibir respuesta del upstream

    # Buffering — para archivos pequeños está bien activado
    proxy_buffering on;
    proxy_buffer_size        4k;
    proxy_buffers            8 4k;

    # Para uploads grandes desactivarlo
    # location /upload/ {
    #     proxy_buffering off;
    #     proxy_pass http://fastapi_app;
    # }

    # Gzip compression
    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain application/json application/javascript text/css;
}
```

---

## Leer la IP real del cliente en FastAPI

Nginx añade `X-Forwarded-For`, pero FastAPI necesita estar configurado para leerla:

```python
# main.py
from fastapi import FastAPI
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app = FastAPI()

# Confiar en los headers de Nginx
# trusted_hosts="127.0.0.1" = solo confiar en nginx (127.0.0.1)
app.add_middleware(ProxyHeadersMiddleware, trusted_hosts="127.0.0.1")

# Ahora request.client.host devuelve la IP real del usuario
```

---

## Comandos útiles

```bash
# Verificar sintaxis
sudo nginx -t

# Recargar config sin reiniciar (sin cortar conexiones activas)
sudo systemctl reload nginx

# Reiniciar completamente
sudo systemctl restart nginx

# Ver logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Ver solo errores
sudo grep -E "error|warn" /var/log/nginx/error.log | tail -50

# Estado del proceso
sudo systemctl status nginx
```

---

## Config en Docker Compose

Cuando la app también corre en Docker:

```nginx
# nginx.conf (para usar dentro del container)
upstream fastapi_app {
    # Nombre del servicio en docker-compose, no 127.0.0.1
    server fastapi:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://fastapi_app;
        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - fastapi

  fastapi:
    build: .
    expose:
      - "8000" # solo exponer internamente, no al host
```
