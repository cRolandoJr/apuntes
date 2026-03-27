[[Automatización y Logs]]

# Cron — Programador de Tareas

## Sintaxis

```
 ┌───────────── minuto (0 - 59)
 │ ┌───────────── hora (0 - 23)
 │ │ ┌───────────── día del mes (1 - 31)
 │ │ │ ┌───────────── mes (1 - 12)
 │ │ │ │ ┌───────────── día de la semana (0 - 6) (Domingo=0 o 7)
 │ │ │ │ │
 * * * * * comando_a_ejecutar
```

## Ejemplos Comunes

```bash
# Backup todos los días a las 3:00 AM
0 3 * * * /home/user/scripts/backup.sh

# Ejecutar cada 5 minutos
*/5 * * * * /comando/a/ejecutar

# Cada hora en punto
0 * * * * /ruta/script.sh

# Lunes a viernes a las 8 AM
0 8 * * 1-5 /ruta/script.sh

# Primer día de cada mes a medianoche
0 0 1 * * /ruta/script.sh
```

## Comandos de Crontab

```bash
# Editar la crontab del usuario actual
crontab -e

# Editar la crontab de root
sudo crontab -e

# Ver las tareas programadas
crontab -l

# Eliminar TODAS las tareas del usuario (cuidado)
crontab -r
```

---

## Ejemplo Práctico: Script de Monitoreo de RAM

### Paso 1: Crear el script

Los scripts del administrador van en `/usr/local/bin/`:

```bash
sudo nano /usr/local/bin/monitor_ram.sh
```

```bash
#!/bin/bash
# Obtiene la fecha actual
FECHA=$(date '+%Y-%m-%d %H:%M:%S')
# Obtiene la RAM libre en MB
RAM_LIBRE=$(free -m | grep Mem | awk '{print $4}')

# Escribe en un log
echo "[$FECHA] RAM Libre: ${RAM_LIBRE} MB" >> /var/log/monitor_ram.log
```

### Paso 2: Dar permisos de ejecución

```bash
sudo chmod +x /usr/local/bin/monitor_ram.sh
```

### Paso 3: Probar manualmente

```bash
sudo /usr/local/bin/monitor_ram.sh
cat /var/log/monitor_ram.log
```

### Paso 4: Programar en crontab

```bash
sudo crontab -e
```

Agregar al final:

```bash
* * * * * /usr/local/bin/monitor_ram.sh
```

### Paso 5: Verificar que funciona

```bash
tail -f /var/log/monitor_ram.log
```

> Para detenerlo: comentar la línea en crontab agregándole `#` al principio.

---

## Troubleshooting

Si tu log está vacío, verificar si Cron ejecutó la tarea:

```bash
# En Debian/Ubuntu
grep CRON /var/log/syslog

# O usando journalctl
journalctl -u cron
```

**Causas comunes de falla:**

- El script no tiene permisos de ejecución (`chmod +x`)
- La ruta del script no es absoluta (Cron no usa tu PATH)
- Variables de entorno faltantes (Cron tiene un entorno mínimo)
