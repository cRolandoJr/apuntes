[[0. General Tips]]

# Systemd y Procesos

## 1. Ciclo de Vida de un Servicio

```bash
# Arrancar un servicio
sudo systemctl start mi-servicio

# Detener un servicio
sudo systemctl stop mi-servicio

# Reiniciar (detiene y vuelve a iniciar)
sudo systemctl restart mi-servicio

# Recargar configuración sin detener (si el servicio lo soporta)
sudo systemctl reload mi-servicio

# Habilitar para que arranque al boot
sudo systemctl enable mi-servicio

# Habilitar Y arrancar en un solo comando
sudo systemctl enable --now mi-servicio

# Deshabilitar (no arranca en el boot)
sudo systemctl disable mi-servicio

# Ver estado actual y últimas líneas de log
sudo systemctl status mi-servicio

# Listar todos los servicios activos
systemctl list-units --type=service

# Listar todos los servicios (incluidos inactivos)
systemctl list-units --type=service --all
```

> **Pro Tip:** Si modificás un archivo `.service`, ejecutá `sudo systemctl daemon-reload` antes de reiniciar el servicio.

---

## 2. Gestión de Procesos

- **Observar procesos:** [[Revisión y visualización de procesos]] (top, htop, ps)
- **Matar y controlar procesos:** [[Manejo y eliminación de procesos]] (kill, signals, jobs)

**Referencia rápida:**

| Señal        | Comando       | Efecto                                              |
| ------------ | ------------- | --------------------------------------------------- |
| SIGTERM (15) | `kill PID`    | Cierre limpio (permite guardar datos)               |
| SIGKILL (9)  | `kill -9 PID` | Destrucción inmediata (solo si SIGTERM no funciona) |
| SIGHUP (1)   | `kill -1 PID` | Recargar configuración                              |

---

## 3. Logs con Journalctl

Systemd guarda los logs en formato binario. Se consultan con `journalctl`:

```bash
# Ver logs de un servicio en tiempo real
journalctl -u nginx -f

# Ver errores recientes del sistema
journalctl -xe

# Filtrar por prioridad (0=emerg ... 3=err ... 7=debug)
journalctl -p err

# Logs desde el último boot
journalctl -b

# Logs de boots anteriores
journalctl --list-boots          # Ver lista de boots
journalctl -b -1                 # Boot anterior

# Logs de los últimos 30 minutos
journalctl --since "30 min ago"

# Espacio que ocupan los logs
journalctl --disk-usage

# Limpiar logs viejos (mantener solo los últimos 7 días)
sudo journalctl --vacuum-time=7d
```

---

**Relacionado:** [[Automatización y Logs]] | [[Cron]] | [[Systemd.Timers]]
