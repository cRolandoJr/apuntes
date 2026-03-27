[[Redes & Troubleshooting]]

# Nginx — Servidor Web y Reverse Proxy

## 1. Instalación y Estado

```bash
sudo apt update && sudo apt install nginx -y
```

```bash
systemctl status nginx
```

**Verificación:** Abrir el navegador y poner la IP del servidor (ej: `http://192.168.x.x`). Si aparece "Welcome to nginx!", funciona.

---

## 2. Configurar Reverse Proxy

Crear el archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/mi-app
```

### Ejemplo: Proxy a una app Go en el puerto 8080

```nginx
server {
    # 1. La Puerta de Entrada
    listen 80;
    listen [::]:80;

    # 2. A quién respondemos (Tu IP o Dominio)
    # Si no tienes dominio, pon la IP de tu servidor o "_" (que significa "cualquiera")
    server_name _;

    # 3. La Redirección (El Proxy)
    location / {
        # Aquí ocurre la magia: Nginx pasa la patata caliente a tu app de Go
        proxy_pass http://localhost:8080;

        # Cabeceras importantes (para que Go sepa quién es el cliente real)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 3. Arquitectura de Configuración (Debian)

- **`sites-available/`** — Borradores de configuración. Pueden existir muchos, pero Nginx los ignora si no están activados.
- **`sites-enabled/`** — Sitios activos. Nginx sirve estos.

No se copian archivos de una carpeta a otra. Se crea un **enlace simbólico**:

## Activar el sitio

```bash
sudo ln -s /etc/nginx/sites-available/mi-app /etc/nginx/sites-enabled/
```

**Desactivar el sitio por defecto** (recomendado para evitar conflictos):

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## 4. Verificar y Aplicar Configuración

**SIEMPRE** verificar la sintaxis antes de recargar:

```bash
sudo nginx -t
```

Buscar: `syntax is ok` y `test is successful`.

Si el test pasa, recargar:

```bash
sudo systemctl reload nginx
```

> **Nunca** reiniciar Nginx sin verificar la sintaxis primero. Un error de tipeo puede tumbar el servidor web completo.

---

## 5. Troubleshooting y Logs

Si recibes `502 Bad Gateway` o `500 Internal Server Error`:

```bash
# Ver errores de Nginx
sudo tail -f /var/log/nginx/error.log

# Ver quién visita tu sitio
sudo tail -f /var/log/nginx/access.log
```

---

## 6. Firewall

Si Nginx corre pero no podés acceder desde el navegador, probablemente el firewall bloquea los puertos:

```bash
sudo ufw allow 'Nginx Full'   # Abre puertos 80 y 443
```
