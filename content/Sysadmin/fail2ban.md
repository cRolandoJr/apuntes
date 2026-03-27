[[SSH]]

# Fail2Ban — Protección contra Fuerza Bruta

Fail2Ban es un demonio que lee los logs de tus servicios (como el journal de systemd) en tiempo real. Si detecta que una misma IP está fallando demasiadas veces en poco tiempo, habla directamente con tu firewall y crea una regla dinámica para bloquear esa IP temporalmente.

---

## 1. Instalación

Con una simple línea de comandos obtenemos el paquete:

```bash
sudo pacman -S fail2ban     # Arch
# sudo apt install fail2ban   # Debian/Ubuntu
```

---

## 2. La Regla de Oro: El archivo `jail.local`

Fail2Ban trae `jail.conf` por defecto. **Nunca editar ese archivo** — las actualizaciones lo sobrescriben. Se crea una copia con prioridad:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

---

## 3. Configurar la Prisión (Jail) de SSH

Abrir el archivo y buscar la sección `[sshd]`:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 45678              # Si cambiaste el puerto SSH
filter = sshd
logpath = /var/log/auth.log   # O si usás systemd puro: backend = systemd
maxretry = 3              # A los 3 intentos fallidos, ban
maxretry_interval = 600   # Ventana de 10 minutos
bantime = 1h              # Bloqueado por 1 hora
```

---

## 4. Activar el Servicio

Ya con la configuración lista, le decimos a systemd que arranque el servicio y lo habilite para que inicie automáticamente cada vez que el servidor se reinicie:

```bash
sudo systemctl enable --now fail2ban
```

---

## 5. Ver la Lista Negra

```bash
# Resumen general (cuántas prisiones activas)
sudo fail2ban-client status

# Ver IPs baneadas en SSH
sudo fail2ban-client status sshd

# Desbanear una IP manualmente
sudo fail2ban-client set sshd unbanip 192.168.1.100
```
