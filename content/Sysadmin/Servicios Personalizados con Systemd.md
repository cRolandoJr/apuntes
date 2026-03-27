[[0. General Tips]]

# Servicios Personalizados con Systemd

## 1. Estructura de un Archivo .service

Los archivos de servicio van en `/etc/systemd/system/` y tienen 3 secciones:

```ini
[Unit]
Description=Mi Servicio Personalizado
Documentation=https://mi-servicio.com/docs
After=network.target          # Iniciar después de la red
Wants=network-online.target   # Quiere red (pero no falla si no hay)
# Requires=postgresql.service # Requiere otro servicio (falla si no arranca)

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/opt/mi-app
ExecStart=/opt/mi-app/bin/servidor
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 2. Tipos de Servicio (Type=)

| Tipo      | Cuándo usarlo                                                                                 |
| --------- | --------------------------------------------------------------------------------------------- |
| `simple`  | El proceso principal es el que se ejecuta en ExecStart (el más común)                         |
| `exec`    | Como simple, pero systemd espera a que exec() sea exitoso                                     |
| `forking` | El proceso hace fork y el padre termina (daemons clásicos, como nginx sin `-g 'daemon off;'`) |
| `oneshot` | Para scripts que se ejecutan una vez y terminan                                               |
| `notify`  | El servicio notifica a systemd cuando está listo (sd_notify)                                  |
| `idle`    | Espera a que otros trabajos terminen antes de arrancar                                        |

```ini
# Ejemplo: script que se ejecuta una vez
[Service]
Type=oneshot
ExecStart=/usr/local/bin/limpiar-cache.sh
RemainAfterExit=yes    # Marcar como "activo" después de terminar

# Ejemplo: daemon clásico con fork
[Service]
Type=forking
PIDFile=/run/mi-daemon.pid
ExecStart=/usr/sbin/mi-daemon --daemon
```

---

## 3. Directivas de [Service]

### Ejecución

```ini
ExecStart=/ruta/al/comando --opciones
ExecStartPre=/usr/local/bin/verificar-config.sh    # Antes de iniciar
ExecStartPost=/usr/local/bin/notificar-inicio.sh   # Después de iniciar
ExecStop=/bin/kill -SIGTERM $MAINPID               # Cómo parar
ExecReload=/bin/kill -SIGHUP $MAINPID              # Cómo recargar config
```

### Reinicio Automático

```ini
Restart=on-failure     # Reiniciar solo si falla (exit code != 0)
# Opciones: no, always, on-success, on-failure, on-abnormal, on-abort, on-watchdog
RestartSec=5           # Esperar 5 segundos antes de reiniciar
StartLimitIntervalSec=300   # Ventana de tiempo para contar reinicios
StartLimitBurst=5           # Máximo N reinicios en la ventana (después no intenta más)
```

### Variables de Entorno

```ini
# Directamente en el .service
Environment=NODE_ENV=production
Environment=PORT=3000

# Desde un archivo
EnvironmentFile=/etc/mi-app/config.env
# El archivo config.env contiene:
# NODE_ENV=production
# DB_HOST=localhost
# DB_PORT=5432
```

### Usuarios y Permisos

```ini
User=appuser
Group=appgroup
# Si necesita crear el usuario automáticamente:
DynamicUser=yes
```

---

## 4. Hardening de Servicios

Limitar lo que un servicio puede hacer (defense in depth):

```ini
[Service]
# Filesystem
ProtectSystem=strict         # /usr, /boot, /etc en solo lectura
ProtectHome=yes              # /home, /root, /run/user inaccesibles
ReadWritePaths=/var/lib/mi-app /var/log/mi-app
PrivateTmp=yes               # /tmp aislado para este servicio

# Seguridad
NoNewPrivileges=yes          # No puede obtener nuevos privilegios
PrivateDevices=yes           # No acceso a dispositivos
ProtectKernelTunables=yes    # No puede cambiar sysctl
ProtectKernelModules=yes     # No puede cargar módulos
ProtectControlGroups=yes     # No puede modificar cgroups

# Red
# RestrictAddressFamilies=AF_INET AF_INET6   # Solo IPv4/IPv6
# PrivateNetwork=yes         # Sin acceso a red (para servicios locales)

# Capacidades (capabilities)
CapabilityBoundingSet=CAP_NET_BIND_SERVICE   # Solo bind a puertos < 1024
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### Ver el score de seguridad de un servicio

```bash
systemd-analyze security mi-servicio.service
# Muestra un score de 0 (inseguro) a 10 (seguro) con sugerencias
```

---

## 5. Ejemplos Completos

### App Node.js

```ini
# /etc/systemd/system/mi-app-node.service
[Unit]
Description=Mi Aplicación Node.js
After=network.target

[Service]
Type=simple
User=nodeapp
WorkingDirectory=/opt/mi-app
ExecStart=/usr/bin/node /opt/mi-app/server.js
Restart=on-failure
RestartSec=10
EnvironmentFile=/etc/mi-app/config.env

# Hardening
ProtectSystem=strict
ReadWritePaths=/var/log/mi-app
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=mi-app-node

[Install]
WantedBy=multi-user.target
```

### App Go (binario compilado)

```ini
# /etc/systemd/system/mi-app-go.service
[Unit]
Description=API Go
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=goapp
ExecStart=/opt/mi-api/bin/api-server
Restart=always
RestartSec=5
EnvironmentFile=/etc/mi-api/config.env

ProtectSystem=strict
ReadWritePaths=/var/log/mi-api
ProtectHome=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

### Script de Mantenimiento (oneshot con timer)

```ini
# /etc/systemd/system/limpiar-tmp.service
[Unit]
Description=Limpiar archivos temporales viejos

[Service]
Type=oneshot
ExecStart=/usr/bin/find /tmp -type f -mtime +7 -delete

# /etc/systemd/system/limpiar-tmp.timer
[Unit]
Description=Ejecutar limpieza de /tmp cada día

[Timer]
OnCalendar=daily
Persistent=true      # Ejecutar si se perdió la última ejecución

[Install]
WantedBy=timers.target
```

```bash
# Activar el timer
sudo systemctl enable --now limpiar-tmp.timer

# Ver timers activos
systemctl list-timers
```

---

## 6. Workflow Completo

```bash
# 1. Crear el archivo de servicio
sudo vim /etc/systemd/system/mi-servicio.service

# 2. Recargar systemd (OBLIGATORIO después de crear/modificar .service)
sudo systemctl daemon-reload

# 3. Iniciar el servicio
sudo systemctl start mi-servicio

# 4. Verificar estado
sudo systemctl status mi-servicio

# 5. Ver logs
sudo journalctl -u mi-servicio -f

# 6. Si funciona, habilitar para que arranque con el sistema
sudo systemctl enable mi-servicio

# Si modificás el .service después:
sudo systemctl daemon-reload
sudo systemctl restart mi-servicio
```

---

## 7. Troubleshooting

```bash
# Ver por qué falló
sudo systemctl status mi-servicio
sudo journalctl -u mi-servicio --no-pager -n 50

# Ver las propiedades aplicadas
systemctl show mi-servicio

# Verificar sintaxis del archivo
systemd-analyze verify /etc/systemd/system/mi-servicio.service

# Si el servicio reinicia en loop
sudo systemctl reset-failed mi-servicio

# Forzar parada de un servicio atascado
sudo systemctl kill mi-servicio
```

---

## 8. Socket Activation (Avanzado)

El servicio solo se inicia cuando alguien se conecta al puerto.

```ini
# /etc/systemd/system/mi-app.socket
[Unit]
Description=Socket para Mi App

[Socket]
ListenStream=8080
Accept=no

[Install]
WantedBy=sockets.target
```

```ini
# /etc/systemd/system/mi-app.service
[Unit]
Description=Mi App (activada por socket)
Requires=mi-app.socket

[Service]
Type=simple
ExecStart=/opt/mi-app/bin/servidor
```

```bash
# Habilitar el socket (no el servicio)
sudo systemctl enable --now mi-app.socket
# El servicio arranca automáticamente cuando alguien conecta al puerto 8080
```
