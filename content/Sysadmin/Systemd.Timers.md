[[Automatización y Logs]]

Aquí es donde entra el SysAdmin Moderno. Systemd separa el "Qué" (Service) del "Cuándo" (Timer).

Sí, son dos archivos en vez de una línea. Pero gana en control y potencia.

#### Paso A: El "Qué" (Service Unit)

Le decimos a Systemd _qué_ comando ejecutar.
```bash
sudo vim /etc/systemd/system/monitor-ram.service
```

```bash
[Unit]
Description=Loguear uso de RAM (Script de una sola vez)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor_ram.sh
```
Nota: `Type=oneshot` significa "ejecútalo, espera que termine y apágate". No es un servicio que se queda corriendo siempre (como Nginx).

#### Paso B: El "Cuándo" (Timer Unit)

Le decimos a Systemd _cuándo_ activar el servicio anterior.

Crea el archivo del timer (debe tener el mismo nombre pero terminar en `.timer`):
```bash
sudo vim /etc/systemd/system/monitor-ram.timer
```

```bash
[Unit]
Description=Correr monitor de RAM cada 1 minuto

[Timer]
# Ejecutar 1 minuto después de que arranque el sistema
OnBootSec=1min
# Y luego ejecutar repetidamente cada 1 minuto
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

**Variante: Ejecución por Calendario (Como Cron)** Si quieres fechas específicas en lugar de intervalos, usa `OnCalendar` en el archivo `.timer`:
```bash
[Timer]
# Ejecutarse diariamente a las 05:00 AM
OnCalendar=*-*-* 05:00:00
# Ejecutarse cada Lunes a las 3 AM
OnCalendar=Mon *-*-* 03:00:00
```
#### Paso C: Activar el Mecanismo

A diferencia de Cron, aquí hay que "darle cuerda" al reloj.
1. Recarga Systemd para que lea los archivos nuevos:
```bash
sudo systemctl daemon-reload
```

2. Inicia el Timer (NO el servicio):
```bash
sudo systemctl start monitor-ram.timer
sudo systemctl enable monitor-ram.timer
```
3. Verifica tus timers activos:
```bash
systemctl list-timers --all
```

4. La magia de systemd timers, verificar los logs!
```bash
sudo journalctl -u monitor-ram.service
```
   