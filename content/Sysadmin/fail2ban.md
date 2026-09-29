[[SSH]]

# Fail2Ban — Protección contra Fuerza Bruta

Fail2Ban es un demonio que lee los logs de tus servicios (como el journal de systemd) en tiempo real. Si detecta que una misma IP está fallando demasiadas veces en poco tiempo, habla directamente con tu firewall y crea una regla dinámica para bloquear esa IP temporalmente.

---

## 1. Instalación

Con una simple línea de comandos obtenemos el paquete:

```bash
sudo apt install fail2ban    # Debian/Ubuntu
# sudo pacman -S fail2ban    # Arch
```

---

## 2. La Regla de Oro: El archivo `jail.local`

Fail2Ban trae `jail.conf` por defecto. **Nunca editar ese archivo** — las actualizaciones lo sobrescriben. Lo que se lee después y tiene prioridad es `jail.local`:

```bash
sudo nano /etc/fail2ban/jail.local     # crearlo vacío y poner SOLO lo que cambiás
```

> **No copiar `jail.conf` entero.** Es tentador (`cp jail.conf jail.local`) pero
> congela los defaults del día que lo copiaste: cuando el paquete actualice
> `jail.conf` con filtros nuevos o corregidos, tu copia los tapa y no te enterás.
> `jail.local` se lee *encima* de `jail.conf`, así que con escribir las 5 líneas que
> cambiás alcanza — todo lo demás lo heredás y se sigue actualizando.

---

## 3. Configurar la Prisión (Jail) de SSH

Abrir el archivo y buscar la sección `[sshd]`:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
# OJO: fail2ban (ConfigParser) NO admite comentarios inline — el "# ..." se
# concatena al valor y rompe la jail. Los comentarios van en su propia línea.
[sshd]
enabled = true
# port: poné el que tengas si cambiaste el SSH (default 22)
port = 45678
filter = sshd
# logpath para auth.log; con systemd puro usar en su lugar: backend = systemd
logpath = /var/log/auth.log
# maxretry: intentos fallidos antes del ban
maxretry = 3
# findtime: ventana de tiempo en que cuentan esos intentos (600s = 10 min)
findtime = 600
# bantime: cuánto dura el bloqueo
bantime = 1h
# banaction: con qué firewall bloquea. En Ubuntu/Debian con ufw activo, poner ufw
# (si no, fail2ban crea sus propias cadenas de iptables, en paralelo a las de ufw)
banaction = ufw
```

> **Nota:** la directiva de la ventana es `findtime`, NO `maxretry_interval`
> (que no existe). Con `maxretry`/`findtime`: "3 fallos en 10 min → ban de 1h".

> **`banaction = ufw` cuando tenés [[ufw]].** Por defecto fail2ban banea con su propia
> acción de iptables, que funciona pero crea cadenas aparte: los bans **no** aparecen
> en `ufw status` y terminás con dos fuentes de verdad para el mismo firewall. Con
> `banaction = ufw` los bans se insertan como reglas de ufw y ves todo en un lado.
> Requiere que exista `/etc/fail2ban/action.d/ufw.conf` (viene en el paquete de Debian
> y Ubuntu).

> **`logpath` vs `backend`.** `/var/log/auth.log` existe si el sistema tiene `rsyslog`
> — Ubuntu **Server** lo trae, pero las imágenes cloud/minimal y varias distros con
> systemd puro no. Si no existe el archivo, en vez de `logpath` va
> `backend = systemd`, que lee el journal directamente. Verificar antes:
> `ls -la /var/log/auth.log`.

> **Se solapa con `ufw limit`, no lo reemplaza.** `limit` corta por *cantidad de
> conexiones* (6 en 30 s) sin mirar si autenticaron; fail2ban banea por *fallos de
> autenticación* leídos del log. Tener los dos está bien: son criterios distintos, y
> fail2ban además te deja el rastro de quién intentó qué —material de valor si después
> mandás esos logs a un SIEM.

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
