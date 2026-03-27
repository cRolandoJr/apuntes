[[0. General Tips]]

# Certificados SSL/TLS

## 1. Conceptos

```
HTTP  → Puerto 80  → Texto plano (cualquiera puede leer el tráfico)
HTTPS → Puerto 443 → Encriptado con TLS (nadie puede interceptar)
```

**Cadena de confianza:**

```
Root CA (Certificate Authority)
  └── Intermediate CA
      └── Tu certificado (para tu dominio)
```

**Archivos típicos:**

| Archivo         | Contenido                               |
| --------------- | --------------------------------------- |
| `privkey.pem`   | Clave privada (NUNCA compartir)         |
| `cert.pem`      | Tu certificado                          |
| `chain.pem`     | Certificados intermedios                |
| `fullchain.pem` | cert.pem + chain.pem (lo que usa Nginx) |

---

## 2. Let's Encrypt con Certbot (Gratuito)

### Instalación

```bash
# Debian/Ubuntu
sudo apt install certbot python3-certbot-nginx

# Si usás Nginx, certbot lo configura automáticamente
```

### Obtener Certificado

```bash
# Automático (Certbot configura Nginx solo)
sudo certbot --nginx -d tudominio.com -d www.tudominio.com

# Solo obtener el certificado (sin modificar Nginx)
sudo certbot certonly --nginx -d tudominio.com

# Si no tenés servidor web corriendo (usa puerto 80 temporal)
sudo certbot certonly --standalone -d tudominio.com

# Modo DNS (para wildcards o si no tenés puerto 80 expuesto)
sudo certbot certonly --manual --preferred-challenges dns -d "*.tudominio.com"
```

### Los certificados se guardan en

```
/etc/letsencrypt/live/tudominio.com/
├── privkey.pem     ← Clave privada
├── cert.pem        ← Certificado
├── chain.pem       ← Cadena intermedia
└── fullchain.pem   ← Certificado + cadena (usar este en Nginx)
```

### Renovación Automática

```bash
# Let's Encrypt expira en 90 días. Certbot instala un timer que renueva:
sudo systemctl status certbot.timer

# Probar la renovación (sin renovar realmente)
sudo certbot renew --dry-run

# Forzar renovación
sudo certbot renew

# Ver certificados instalados
sudo certbot certificates
```

---

## 3. Configurar Nginx con TLS

```nginx
server {
    listen 80;
    server_name tudominio.com www.tudominio.com;

    # Redireccionar todo HTTP a HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name tudominio.com www.tudominio.com;

    # Certificados
    ssl_certificate     /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;

    # Seguridad TLS
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';

    # Headers de seguridad
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# Verificar sintaxis y recargar
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Certificados Autofirmados (para entornos internos)

```bash
# Generar clave privada + certificado en un solo comando (válido por 365 días)
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key \
  -out /etc/ssl/certs/selfsigned.crt \
  -subj "/CN=mi-servidor-interno"
```

> Los navegadores marcarán como "no seguro" pero la conexión sí está encriptada. Útil para paneles internos, APIs internas, etc.

---

## 5. Comandos OpenSSL Útiles

```bash
# Ver información de un certificado
openssl x509 -in cert.pem -text -noout

# Ver fecha de expiración
openssl x509 -in cert.pem -enddate -noout

# Verificar que certificado y clave coinciden
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa -noout -modulus -in privkey.pem | md5sum
# Si los hashes son iguales, coinciden

# Verificar el certificado de un sitio remoto
openssl s_client -connect tudominio.com:443 -servername tudominio.com

# Ver la cadena completa de certificados
openssl s_client -connect tudominio.com:443 -showcerts

# Generar un CSR (Certificate Signing Request) para una CA comercial
openssl req -new -newkey rsa:2048 -nodes \
  -keyout privkey.pem \
  -out request.csr \
  -subj "/C=AR/ST=Rio Negro/L=Viedma/O=MiEmpresa/CN=tudominio.com"
```

---

## 6. Troubleshooting

```bash
# El certificado expiró
sudo certbot renew
sudo systemctl reload nginx

# Error "certificate not trusted" → Falta la cadena intermedia
# Usar fullchain.pem en vez de cert.pem en Nginx

# Error "certificate mismatch" → El certificado no es para ese dominio
openssl x509 -in cert.pem -text -noout | grep "Subject:"

# Verificar que el puerto 443 está abierto
sudo ss -tulpn | grep 443
sudo ufw allow 443/tcp
```
