# Fase 2: SysAdmin Profesional — Meses 4 a 6

> **Objetivo**: Administrar servidores Linux como un profesional. Servicios, storage, backups, SSH, seguridad, y automatización básica. Al terminar esta fase, podrías administrar servidores de producción.

---

## Mes 4: Servicios y Systemd

### 4.1 ¿Qué es systemd?

Systemd es el **sistema de inicio y gestión de servicios** de Linux moderno. Es el PID 1 — el primer proceso que arranca y el padre de todos los demás.

**¿Qué hace systemd?**

- Arranca el sistema (boot)
- Gestiona servicios (iniciar, parar, reiniciar, habilitar al arranque)
- Gestiona logs (journald)
- Gestiona timers (reemplazo de cron)
- Gestiona mount points, sockets, targets...

### 4.2 Unidades (Units)

Todo en systemd es una **unidad** (unit). Los tipos más comunes:

| Tipo    | Extensión  | Propósito                                |
| ------- | ---------- | ---------------------------------------- |
| Service | `.service` | Un servicio/daemon (nginx, sshd, tu API) |
| Timer   | `.timer`   | Tarea programada (reemplazo de cron)     |
| Socket  | `.socket`  | Activación por socket                    |
| Mount   | `.mount`   | Punto de montaje                         |
| Target  | `.target`  | Agrupación de unidades (como runlevels)  |

### 4.3 Gestión de Servicios

```bash
# Estado de un servicio
systemctl status nginx
# Active: active (running) → está corriendo
# Active: inactive (dead)  → está parado
# Active: failed           → crasheó

# Controlar servicios
sudo systemctl start nginx      # Iniciar
sudo systemctl stop nginx       # Parar
sudo systemctl restart nginx    # Parar + iniciar
sudo systemctl reload nginx     # Recargar config sin parar (no todos soportan)
sudo systemctl enable nginx     # Iniciar automáticamente al boot
sudo systemctl disable nginx    # No iniciar al boot
sudo systemctl enable --now nginx  # Habilitar E iniciar

# Ver todos los servicios
systemctl list-units --type=service
systemctl list-units --type=service --state=running

# Ver servicios que fallaron
systemctl --failed
```

### 4.4 Crear tu Propio Servicio

Vamos a hacer un servicio systemd para tu API de productos:

```ini
# /etc/systemd/system/gestion-productos.service
[Unit]
Description=API GraphQL de Gestión de Productos
After=network.target
# "After=network.target" → esperar a que la red esté lista antes de arrancar

[Service]
Type=simple
# "simple" → systemd considera que el servicio está listo cuando arranca el proceso

User=apiuser
Group=apiuser
# Corre bajo un usuario dedicado, NO root (principio de mínimo privilegio)

WorkingDirectory=/opt/gestion_productos
ExecStart=/opt/gestion_productos/gestion_productos
# El binario a ejecutar

Restart=on-failure
RestartSec=5
# Si el proceso muere por un error, reiniciar después de 5 segundos

StandardOutput=journal
StandardError=journal
# Los logs van a journald (se ven con journalctl)

Environment=PORT=8081
# Variables de entorno

[Install]
WantedBy=multi-user.target
# Se activa cuando el sistema llega a modo multi-usuario (boot normal)
```

```bash
# Cargar el servicio
sudo systemctl daemon-reload        # Recargar configs de systemd
sudo systemctl enable --now gestion-productos

# Ver logs
journalctl -u gestion-productos -f  # -f = follow (tiempo real)
journalctl -u gestion-productos --since "1 hour ago"
journalctl -u gestion-productos --since today --no-pager
```

### 4.5 Timers (Reemplazo de Cron)

```ini
# /etc/systemd/system/health-check.service
[Unit]
Description=Health Check del Sistema

[Service]
Type=oneshot
ExecStart=/usr/local/bin/health_check.sh
```

```ini
# /etc/systemd/system/health-check.timer
[Unit]
Description=Ejecutar Health Check cada 5 minutos

[Timer]
OnCalendar=*:0/5
# Cada 5 minutos (en minutos 0, 5, 10, 15, 20...)
Persistent=true
# Si el sistema estaba apagado cuando tocaba ejecutar, hacerlo al arrancar

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now health-check.timer
systemctl list-timers  # Ver todos los timers activos
```

### 4.6 Journalctl — El Sistema de Logs

```bash
# Logs del sistema completo
journalctl -b               # Desde el último boot
journalctl -b -1             # Desde el boot anterior
journalctl --since "2 hours ago"
journalctl --since "2026-03-23 10:00" --until "2026-03-23 12:00"

# Logs de un servicio
journalctl -u sshd           # Logs de SSH
journalctl -u nginx -f       # Seguir en tiempo real

# Filtrar por prioridad
journalctl -p err            # Solo errores y más graves
# Prioridades: emerg, alert, crit, err, warning, notice, info, debug

# Logs del kernel
journalctl -k                # Mensajes del kernel (dmesg)

# Espacio que ocupan los logs
journalctl --disk-usage
sudo journalctl --vacuum-size=500M  # Limitar a 500MB
```

**Ejercicios verificables**:

```bash
# 1. Crear el servicio de tu API
sudo useradd -r -s /usr/sbin/nologin apiuser
# -r = usuario de sistema, -s /usr/sbin/nologin = no puede loguearse

# 2. Compilar y deployar
cd /home/Rolando/gestion_productos
go build -o /tmp/gestion_productos ./cmd/main.go
sudo mkdir -p /opt/gestion_productos
sudo cp /tmp/gestion_productos /opt/gestion_productos/
sudo chown -R apiuser:apiuser /opt/gestion_productos

# 3. Crear el unit file (el .service de arriba)
# 4. Iniciar y verificar
sudo systemctl daemon-reload
sudo systemctl start gestion-productos
systemctl status gestion-productos
curl http://localhost:8081/

# 5. Ver logs
journalctl -u gestion-productos --no-pager | tail -5

# 6. Simular un crash y verificar el restart
sudo kill -9 $(pgrep gestion_productos)
sleep 6  # Esperar los 5 segundos de RestartSec
systemctl status gestion-productos  # Debería estar running de nuevo
```

---

## Mes 5: Storage, Backups y SSH

### 5.1 Discos y Particiones

```
Disco físico (/dev/sda)
├── Partición 1 (/dev/sda1) → /boot (500MB, ext4)
├── Partición 2 (/dev/sda2) → / (50GB, ext4)
└── Partición 3 (/dev/sda3) → /home (resto, ext4)
```

**Comandos esenciales**:

```bash
# Ver discos y particiones
lsblk                    # Árbol de bloques (visual)
fdisk -l                 # Detalle de discos y particiones
df -h                    # Uso de sistemas de archivos montados
du -sh /var/log          # Tamaño de un directorio

# Crear una partición (en un disco nuevo /dev/sdb)
sudo fdisk /dev/sdb      # Interactivo: n (new), p (primary), w (write)

# Crear filesystem
sudo mkfs.ext4 /dev/sdb1

# Montar
sudo mkdir /mnt/datos
sudo mount /dev/sdb1 /mnt/datos

# Montaje permanente (se mantiene al reiniciar)
# Agregar en /etc/fstab:
# /dev/sdb1  /mnt/datos  ext4  defaults  0  2
```

### 5.2 LVM (Logical Volume Manager)

LVM agrega una capa de abstracción entre los discos físicos y los filesystems. Te permite **redimensionar volúmenes sin downtime**.

```
Discos Físicos     → Physical Volumes (PV)  → Volume Group (VG)  → Logical Volumes (LV)
/dev/sda1 ────────→ PV1 ──┐                                     ┌─→ lv-root (50GB, /)
/dev/sdb1 ────────→ PV2 ──┼──→ vg-principal (200GB total) ──────┤
/dev/sdc1 ────────→ PV3 ──┘                                     └─→ lv-data (150GB, /data)
```

**¿Por qué LVM?** → Podés agregar un disco nuevo al VG y extender un LV que se quedó sin espacio. Sin LVM tendrías que re-particionar y probablemente reiniciar. Con LVM es en caliente.

```bash
# Crear PV
sudo pvcreate /dev/sdb1

# Crear VG
sudo vgcreate vg-datos /dev/sdb1

# Crear LV
sudo lvcreate -L 10G -n lv-app vg-datos

# Formatear y montar
sudo mkfs.ext4 /dev/vg-datos/lv-app
sudo mount /dev/vg-datos/lv-app /opt/app

# Extender un LV (agregar 5GB)
sudo lvextend -L +5G /dev/vg-datos/lv-app
sudo resize2fs /dev/vg-datos/lv-app   # Redimensionar el filesystem
```

### 5.3 Backups

**Regla 3-2-1**: 3 copias, 2 medios diferentes, 1 offsite (en otra ubicación).

**Herramientas**:

```bash
# rsync — La herramienta de backup más versátil
rsync -avz --delete /data/ /backup/data/
# -a = archive (permisos, dueño, fechas)
# -v = verbose
# -z = comprimir durante transferencia
# --delete = borrar en destino lo que ya no está en origen

# rsync remoto (vía SSH)
rsync -avz /data/ usuario@servidor_backup:/backup/data/

# tar — Empaquetar y comprimir
tar czf backup-$(date +%Y%m%d).tar.gz /opt/app/data/
# c=create, z=gzip, f=archivo

# Restaurar
tar xzf backup-20260323.tar.gz -C /opt/app/
# x=extract

# Script de backup automatizado
#!/bin/bash
set -euo pipefail
FECHA=$(date +%Y%m%d_%H%M%S)
ORIGEN="/opt/app/data"
DESTINO="/backup"
LOG="/var/log/backup.log"

echo "[$FECHA] Iniciando backup..." >> "$LOG"
rsync -avz --delete "$ORIGEN/" "$DESTINO/current/" >> "$LOG" 2>&1

# Snapshot comprimido semanal
if [[ $(date +%u) == 7 ]]; then  # Domingo
    tar czf "$DESTINO/weekly/backup-$FECHA.tar.gz" "$DESTINO/current/"
    echo "[$FECHA] Snapshot semanal creado" >> "$LOG"
fi

# Limpiar snapshots de más de 30 días
find "$DESTINO/weekly/" -name "*.tar.gz" -mtime +30 -delete
echo "[$FECHA] Backup completado" >> "$LOG"
```

### 5.4 SSH (Secure Shell) — En Profundidad

SSH es LA herramienta de acceso remoto. Como Platform Engineer, vivirás en SSH.

**Autenticación por clave pública** (nunca uses solo contraseña en producción):

```bash
# 1. Generar par de claves
ssh-keygen -t ed25519 -C "rolando@workstation"
# ed25519 es más seguro y rápido que RSA
# Genera: ~/.ssh/id_ed25519 (privada) y ~/.ssh/id_ed25519.pub (pública)

# 2. Copiar la clave pública al servidor
ssh-copy-id usuario@servidor
# Esto agrega tu clave pública a ~/.ssh/authorized_keys del servidor

# 3. Ahora podés conectarte sin contraseña
ssh usuario@servidor
```

**Configuración del cliente SSH** (`~/.ssh/config`):

```
# ~/.ssh/config — Alias para servidores
Host produccion
    HostName 192.168.100.10
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host staging
    HostName 192.168.100.20
    User deploy
    IdentityFile ~/.ssh/id_ed25519

Host bastion
    HostName bastion.empresa.com
    User rolando
    IdentityFile ~/.ssh/id_ed25519

# Acceder a un servidor DETRÁS del bastion (ProxyJump)
Host interno
    HostName 10.0.1.50
    User admin
    ProxyJump bastion
```

```bash
# Ahora en vez de:
ssh -i ~/.ssh/id_ed25519 deploy@192.168.100.10
# Hacés:
ssh produccion

# Y para llegar al servidor interno a través del bastion:
ssh interno
# SSH se conecta primero al bastion y desde ahí al interno
```

**Hardening de SSH** (seguridad del servidor — `/etc/ssh/sshd_config`):

```
# Cambios críticos de seguridad:
PermitRootLogin no              # NUNCA permitir login como root
PasswordAuthentication no       # Solo claves públicas
PubkeyAuthentication yes        # Habilitar claves públicas
MaxAuthTries 3                  # Máximo 3 intentos
AllowUsers deploy rolando       # Solo estos usuarios pueden entrar
ClientAliveInterval 300         # Desconectar después de 5 min inactivo
ClientAliveCountMax 2           # 2 intentos de keepalive
```

```bash
# Aplicar cambios
sudo systemctl reload sshd

# Verificar ANTES de cerrar tu sesión actual:
# Abrí OTRA terminal y probá que podés conectarte con la nueva config
# Si te quedás afuera por una mala config... sin acceso físico perdiste el servidor
```

**SSH Tunneling** (port forwarding):

```bash
# Local: Acceder a un servicio remoto como si fuera local
ssh -L 5432:localhost:5432 usuario@servidor
# Ahora localhost:5432 en TU PC llega al PostgreSQL del servidor

# Remoto: Exponer un servicio local al servidor remoto
ssh -R 8080:localhost:3000 usuario@servidor
# Ahora servidor:8080 llega a tu localhost:3000

# Dinámico (SOCKS proxy): Navegar a través del servidor
ssh -D 1080 usuario@servidor
# Configurar tu navegador para usar SOCKS proxy en localhost:1080
```

**Ejercicios verificables**:

```bash
# 1. Generar claves SSH
ssh-keygen -t ed25519 -C "rolando@lab"
ls -la ~/.ssh/
# Verificar: id_ed25519 (permiso 600), id_ed25519.pub (permiso 644)

# 2. Crear config SSH
cat ~/.ssh/config
# Verificar: podés conectarte con el alias

# 3. Hacer un túnel SSH local
# En el servidor tenés PostgreSQL en 5432
ssh -L 15432:localhost:5432 usuario@servidor
# En otra terminal:
psql -h localhost -p 15432 -U usuario dbname
# Estás accediendo al PostgreSQL remoto como si fuera local
```

---

## Mes 6: Seguridad y Hardening

### 6.1 Principios de Seguridad

1. **Mínimo privilegio**: Cada proceso/usuario solo tiene los permisos que necesita
2. **Defensa en profundidad**: Múltiples capas de seguridad (no confiar en una sola)
3. **Menor superficie de ataque**: Mientras menos servicios expuestos, menos vectores de ataque
4. **Fail secure**: Ante una falla, el sistema queda en estado seguro (cerrado, no abierto)

### 6.2 Hardening de Linux

```bash
# 1. Actualizaciones automáticas de seguridad
sudo apt install unattended-upgrades  # Debian/Ubuntu
sudo dpkg-reconfigure -plow unattended-upgrades

# 2. Deshabilitar servicios innecesarios
systemctl list-units --type=service --state=running
# ¿Hay servicios que no necesitás? Deshabilitados:
sudo systemctl disable --now cups       # ¿Necesitás imprimir en un servidor?
sudo systemctl disable --now avahi-daemon  # ¿mDNS en un servidor? No.

# 3. Configurar fail2ban (bloquear IPs que intentan fuerza bruta)
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
# Editar jail.local:
# [sshd]
# enabled = true
# maxretry = 3
# bantime = 3600    # 1 hora
# findtime = 600    # En 10 minutos
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd

# 4. Limitar acceso a sudo
# Crear un grupo específico
sudo groupadd ops
sudo usermod -aG ops rolando
# En /etc/sudoers.d/ops:
# %ops ALL=(ALL) /usr/bin/systemctl, /usr/bin/journalctl
# El grupo ops solo puede ejecutar systemctl y journalctl con sudo

# 5. Auditoría: quién hizo qué
sudo apt install auditd
# Monitorear cambios en archivos críticos:
sudo auditctl -w /etc/passwd -p wa -k user_changes
# -w = watch file, -p wa = write+attribute changes, -k = tag
sudo ausearch -k user_changes   # Ver intentos de cambio
```

### 6.3 Certificados TLS (HTTPS)

```bash
# Generar un certificado auto-firmado (para lab/desarrollo)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=localhost"

# Verificar un certificado
openssl x509 -in cert.pem -text -noout | head -20

# Verificar el certificado de un servidor remoto
openssl s_client -connect google.com:443 < /dev/null 2>/dev/null | \
  openssl x509 -text -noout | head -20

# En producción: usar Let's Encrypt (gratuito, automatizado)
# Se gestiona con certbot
sudo apt install certbot
sudo certbot certonly --standalone -d tudominio.com
# Renueva automáticamente cada 90 días
```

### 6.4 Ejercicio Integrador — Hardening de un Servidor

```bash
# Checklist de verificación:
# 1. SSH
sshd -T | grep -E "permitrootlogin|passwordauthentication|pubkeyauthentication"
# Esperado: permitrootlogin no, passwordauthentication no, pubkeyauthentication yes

# 2. Firewall
sudo iptables -L -n | head -20
# Solo puertos necesarios abiertos

# 3. Servicios
systemctl list-units --type=service --state=running | wc -l
# Cuanto menos, mejor. ¿Sabés para qué sirve cada uno?

# 4. Usuarios
awk -F: '$3 == 0' /etc/passwd
# Solo root debería tener UID 0

# 5. Archivos SUID
find / -perm -4000 -type f 2>/dev/null | wc -l
# Conocé cada uno. Si hay alguno raro, investigá.

# 6. Puertos abiertos
ss -tlnp
# ¿Hay algo que no debería estar ahí?

# 7. Fail2ban
sudo fail2ban-client status
# ¿Está activo? ¿Cuántas IPs baneó?

# 8. Logs
journalctl -p err --since "24 hours ago" --no-pager | head -20
# ¿Hay errores que necesiten atención?
```

---

## Proyecto Integrador de Fase 2

### "Servidor de Producción Simulado"

Configurá un servidor (VM o tu PC) como si fuera producción:

1. **Tu API como servicio systemd** operada por el usuario `apiuser`
2. **Timer** que ejecute el health check cada 5 minutos
3. **SSH** con:
   - Solo autenticación por clave pública
   - Root login deshabilitado
   - Config en `~/.ssh/config` con alias
4. **Firewall** que solo permite SSH (22) y la API (8081)
5. **Fail2ban** protegiendo SSH
6. **Backups** automáticos con rsync (aunque sea a otro directorio)
7. **Logs** rotados automáticamente (no crecen infinitamente)
8. **Documentación** de todo lo configurado en un README

**Verificación final**:

```bash
# Todos estos comandos deben dar resultados esperados:
systemctl status gestion-productos    # active (running)
systemctl list-timers | grep health   # timer activo
ss -tlnp | grep -cE "22|8081"       # exactamente 2 líneas
sudo fail2ban-client status sshd     # enabled
ls /backup/                          # hay backups
journalctl --disk-usage              # logs controlados
```

---

## Recursos para esta Fase

### Libros

1. **"UNIX and Linux System Administration Handbook" de Nemeth et al.** (5ta edición) — LA biblia del sysadmin. 1200+ páginas. Si solo leés un libro de sysadmin, que sea este.
2. **"SSH Mastery" de Michael W. Lucas** — Corto, práctico, cubre todo sobre SSH.
3. **"Linux Server Security" de Chris Binnie** — Hardening práctico.

### Cursos y Labs

- **Linux Upskill Challenge** (linuxupskillchallenge.org) — 20 días de práctica real con un servidor Linux, gratis
- **SadServers** (sadservers.com) — Más escenarios de troubleshooting
- **CyberDefenders** (cyberdefenders.org) — Labs de seguridad

### Certificaciones a considerar

- **LPIC-1** (Linux Professional Institute Certification Level 1) — Preparala en esta fase
  - Examen 101 + 102
  - Recursos: wiki.lpi.org tiene los objetivos oficiales gratuitos
  - Libro: "LPIC-1 Study Guide" de Christine Bresnahan

---

## Checkpoint: ¿Estoy listo para la Fase 3?

- [ ] ¿Puedo crear un servicio systemd desde cero?
- [ ] ¿Puedo leer y entender logs con journalctl?
- [ ] ¿Puedo crear un timer systemd?
- [ ] ¿Sé qué es LVM y para qué sirve?
- [ ] ¿Puedo hacer un backup con rsync?
- [ ] ¿Puedo configurar SSH con claves públicas?
- [ ] ¿Puedo hacer SSH tunneling?
- [ ] ¿Puedo explicar los principios de seguridad?
- [ ] ¿Puedo hardening un servidor Linux?
- [ ] ¿Puedo explicar qué hace fail2ban?

Si respondiste 8+ de 10: avanzá a la Fase 3.
