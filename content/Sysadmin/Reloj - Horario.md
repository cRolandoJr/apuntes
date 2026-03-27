[[0. General Tips]]

# Reloj y Horario del Sistema

## 1. Consultar Fecha y Hora

```bash
# Hora del usuario (según timezone configurado)
date

# Hora UTC (la hora del sistema/kernel)
date -u

# Formato personalizado
date '+%Y-%m-%d %H:%M:%S'
```

---

## 2. timedatectl — Gestión Completa del Reloj

```bash
# Ver estado completo (hora local, UTC, timezone, NTP)
timedatectl

# Ver zonas horarias disponibles
timedatectl list-timezones

# Filtrar por región
timedatectl list-timezones | grep America

# Configurar zona horaria
sudo timedatectl set-timezone America/Argentina/Buenos_Aires

# Activar sincronización NTP (reloj automático por internet)
sudo timedatectl set-ntp true

# Desactivar NTP (para setear hora manual)
sudo timedatectl set-ntp false

# Setear hora manualmente (solo con NTP desactivado)
sudo timedatectl set-time '2026-03-25 14:30:00'
```

---

## 3. NTP — Sincronización de Hora por Red

En servidores es **crítico** que la hora sea exacta (logs, certificados TLS, Kerberos, cron).

```bash
# Ver estado del servicio NTP de systemd
systemctl status systemd-timesyncd

# Forzar sincronización
sudo systemctl restart systemd-timesyncd

# Ver con qué servidor NTP está sincronizado
timedatectl timesync-status
```

> **¿Por qué importa?** Si la hora del servidor está desincronizada: los logs no coinciden entre servidores, los certificados TLS pueden fallar, Cron ejecuta tareas a la hora incorrecta, y Kerberos/AD deja de funcionar.

![[Pasted image 20260316185834.png]]
