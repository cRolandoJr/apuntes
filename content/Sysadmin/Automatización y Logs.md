[[0. General Tips]]

# Automatización y Logs

## 1. Programación de Tareas

Linux ofrece dos mecanismos para ejecutar tareas automáticamente:

### [[Cron]] — El clásico

- **Uso:** Scripts rápidos, tareas personales, backups sencillos.
- **Ventaja:** Una sola línea de configuración.
- **Desventaja:** Difícil de debuggear, sin logs nativos, no maneja dependencias (ej: "esperar a que haya red").

### [[Systemd.Timers]] — El moderno

- **Uso:** Tareas de infraestructura, mantenimiento de servidores, entornos de producción.
- **Ventaja:** Logs integrados (`journalctl`), precisión de milisegundos, manejo de dependencias.
- **Estructura:** Requiere dos archivos (`.service` para la acción, `.timer` para la frecuencia).

> **¿Cuándo usar cuál?** Para scripts personales rápidos → Cron. Para servicios en producción → Systemd Timers.

---

## 2. Logs del Sistema

Systemd guarda los logs en formato binario. Se consultan con `journalctl`:

```bash
# Ver logs de un servicio específico en tiempo real
journalctl -u nginx -f

# Ver errores recientes del sistema
journalctl -xe

# Ver logs desde el último boot
journalctl -b

# Ver logs de los últimos 30 minutos
journalctl --since "30 min ago"

# Ver logs de un período específico
journalctl --since "2026-03-20" --until "2026-03-21"
```

Logs tradicionales (archivos de texto) se encuentran en `/var/log/`:

```bash
# Log general del sistema (Debian/Ubuntu)
cat /var/log/syslog

# Log de autenticación (intentos de login SSH, sudo, etc.)
cat /var/log/auth.log

# Seguir un log en tiempo real
tail -f /var/log/syslog
```

---

## 3. Búsqueda y Manipulación de Archivos

- [[grep y find]]
