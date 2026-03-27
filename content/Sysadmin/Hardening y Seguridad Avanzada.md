[[0. General Tips]]

# Hardening y Seguridad Avanzada

## 1. Checklist de Hardening Inicial

Después de instalar un servidor, hacer como mínimo:

```
[x] Actualizar todo: apt update && apt upgrade
[x] Crear usuario no-root con sudo
[x] Configurar SSH con clave pública, deshabilitar password y root login
[x] Configurar firewall (ufw/iptables)
[x] Instalar fail2ban
[ ] Configurar actualizaciones automáticas de seguridad
[ ] Deshabilitar servicios innecesarios
[ ] Configurar PAM y política de contraseñas
[ ] Configurar auditoría (auditd)
[ ] Configurar logrotate
[ ] Revisar permisos SUID/SGID
[ ] Asegurar GRUB con contraseña
```

---

## 2. Actualizaciones Automáticas de Seguridad

```bash
# Instalar
sudo apt install unattended-upgrades

# Activar
sudo dpkg-reconfigure -plow unattended-upgrades

# Configuración: /etc/apt/apt.conf.d/50unattended-upgrades
# Verificar que incluya:
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};

# Verificar que funciona
sudo unattended-upgrades --dry-run --debug
```

---

## 3. Deshabilitar Servicios Innecesarios

```bash
# Ver todos los servicios activos
systemctl list-units --type=service --state=running

# Servicios comúnmente innecesarios en un servidor
sudo systemctl disable --now avahi-daemon      # mDNS (no necesario en servers)
sudo systemctl disable --now cups              # Impresión
sudo systemctl disable --now bluetooth         # Bluetooth
sudo systemctl disable --now ModemManager      # Modems

# Ver puertos abiertos (detectar servicios expuestos)
sudo ss -tlnp
```

---

## 4. PAM — Política de Contraseñas

```bash
# Instalar módulo de calidad de contraseñas
sudo apt install libpam-pwquality

# Configurar: /etc/security/pwquality.conf
minlen = 12          # Mínimo 12 caracteres
dcredit = -1         # Al menos 1 dígito
ucredit = -1         # Al menos 1 mayúscula
lcredit = -1         # Al menos 1 minúscula
ocredit = -1         # Al menos 1 carácter especial
maxrepeat = 3        # No más de 3 caracteres iguales seguidos
reject_username      # No permitir el nombre de usuario en la contraseña
```

### Bloqueo por intentos fallidos

```bash
# En /etc/pam.d/common-auth, agregar ANTES de las otras líneas auth:
auth required pam_faillock.so preauth deny=5 unlock_time=900
auth required pam_faillock.so authfail deny=5 unlock_time=900

# deny=5: bloquear después de 5 intentos fallidos
# unlock_time=900: desbloquear después de 15 minutos

# Ver usuarios bloqueados
faillock --user nombre_usuario

# Desbloquear manualmente
faillock --user nombre_usuario --reset
```

---

## 5. sysctl — Hardening del Kernel

```bash
# Archivo: /etc/sysctl.d/99-hardening.conf

# --- Red ---
# Ignorar pings (protección mínima)
net.ipv4.icmp_echo_ignore_all = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Protección contra SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# No aceptar source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# No aceptar ICMP redirects (previene MITM)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Activar protección contra IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Loggear paquetes "marcianos" (con IPs imposibles)
net.ipv4.conf.all.log_martians = 1

# Deshabilitar IP forwarding (si NO es router/firewall)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# --- Sistema ---
# Restringir dmesg a root
kernel.dmesg_restrict = 1

# Restringir acceso a kernel pointers
kernel.kptr_restrict = 2

# Proteger symlinks y hardlinks
fs.protected_symlinks = 1
fs.protected_hardlinks = 1

# Aplicar cambios
sudo sysctl --system
```

---

## 6. Auditoría con auditd

```bash
# Instalar
sudo apt install auditd

# Iniciar y habilitar
sudo systemctl enable --now auditd

# Agregar reglas de auditoría
# Archivo: /etc/audit/rules.d/custom.rules

# Monitorear cambios en /etc/passwd
-w /etc/passwd -p wa -k identity_changes

# Monitorear cambios en /etc/shadow
-w /etc/shadow -p wa -k identity_changes

# Monitorear cambios en sudoers
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/sudoers.d/ -p wa -k sudo_changes

# Monitorear cambios en SSH
-w /etc/ssh/sshd_config -p wa -k ssh_changes

# Monitorear ejecución de comandos privilegiados
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

# Cargar reglas
sudo augenrules --load

# Consultar logs de auditoría
sudo ausearch -k identity_changes
sudo ausearch -k ssh_changes --start today

# Reporte resumido
sudo aureport --summary
sudo aureport --auth    # Intentos de autenticación
sudo aureport --login   # Logins
```

---

## 7. logrotate — Rotación de Logs

```bash
# Config global: /etc/logrotate.conf
# Configs por servicio: /etc/logrotate.d/

# Ejemplo: /etc/logrotate.d/mi-aplicacion
/var/log/mi-app/*.log {
    daily               # Rotar diariamente
    missingok           # No dar error si no existe
    rotate 14           # Mantener 14 archivos rotados
    compress            # Comprimir con gzip
    delaycompress       # Comprimir a partir de la segunda rotación
    notifempty          # No rotar si está vacío
    create 0640 www-data adm   # Crear nuevo con estos permisos
    sharedscripts       # Ejecutar scripts una sola vez
    postrotate
        systemctl reload mi-app > /dev/null 2>&1 || true
    endscript
}

# Probar configuración (sin ejecutar)
sudo logrotate --debug /etc/logrotate.d/mi-aplicacion

# Forzar rotación manual
sudo logrotate --force /etc/logrotate.d/mi-aplicacion

# Ver estado de rotaciones
cat /var/lib/logrotate/status
```

---

## 8. Buscar Archivos SUID/SGID Sospechosos

Los archivos con bit SUID se ejecutan como root. Un SUID no autorizado es un vector de escalada de privilegios.

```bash
# Buscar todos los archivos SUID
sudo find / -perm -4000 -type f 2>/dev/null

# Buscar todos los archivos SGID
sudo find / -perm -2000 -type f 2>/dev/null

# Lista de SUID normales (no preocupan):
# /usr/bin/sudo, /usr/bin/passwd, /usr/bin/chsh,
# /usr/bin/newgrp, /usr/bin/mount, /usr/bin/umount,
# /usr/bin/su, /usr/bin/pkexec

# Si encontrás algo raro, investigar:
dpkg -S /ruta/al/archivo    # ¿A qué paquete pertenece?
```

---

## 9. Asegurar GRUB

```bash
# Crear contraseña para GRUB
sudo grub-mkpasswd-pbkdf2
# Copiar el hash generado

# Agregar al final de /etc/grub.d/40_custom:
set superusers="admin"
password_pbkdf2 admin grub.pbkdf2.sha512.10000.HASH_COMPLETO

# Regenerar GRUB
sudo update-grub
```

---

## 10. AppArmor (Ubuntu/Debian)

```bash
# Ver estado
sudo aa-status

# Modos:
# enforce   → bloquea violaciones
# complain  → solo loggea violaciones (para testing)
# disabled  → desactivado

# Poner perfil en modo enforce
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx

# Poner en modo complain (testing)
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# Recargar perfiles
sudo systemctl reload apparmor

# Ver logs de violaciones
sudo dmesg | grep apparmor
sudo journalctl -k | grep apparmor
```

---

## 11. Escaneo Básico de Vulnerabilidades

```bash
# Lynis — auditoría de seguridad del sistema
sudo apt install lynis

# Ejecutar auditoría completa
sudo lynis audit system

# El reporte incluye:
# - Puntaje de hardening
# - Sugerencias específicas
# - Warnings y peligros

# Ver sugerencias del último escaneo
sudo grep "suggestion" /var/log/lynis.log
```

---

## 12. Monitoreo de Integridad con AIDE

```bash
# Instalar
sudo apt install aide

# Inicializar base de datos
sudo aideinit

# Copiar base de datos
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Verificar integridad (¿cambió algo?)
sudo aide --check

# Después de cambios autorizados, actualizar la base
sudo aide --update
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

---

## 13. Resumen de Seguridad — Comandos Rápidos

```bash
# ¿Quién está conectado ahora?
who
w

# Últimos logins
last -n 20

# Intentos de login fallidos
sudo lastb -n 20

# Puertos abiertos
sudo ss -tlnp

# Conexiones activas
sudo ss -tnp

# Procesos como root
ps aux | awk '$1 == "root"'

# Archivos modificados en las últimas 24 horas en /etc
sudo find /etc -mtime -1 -type f

# Verificar integridad de paquetes instalados
sudo debsums --changed
```
