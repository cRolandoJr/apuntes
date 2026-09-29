# Fase 1: Linux & Redes — Meses 1 a 3

> **Objetivo**: Dominar Linux a nivel de administración y entender redes TCP/IP en profundidad. Estos son los cimientos de TODO lo que viene después. Un Platform Engineer que no entiende procesos, filesystem o TCP/IP es como un arquitecto que no entiende física.

---

## Mes 1: Linux desde Adentro

### 1.1 El Kernel y el Espacio de Usuario

El kernel de Linux es el programa que controla TODO el hardware. Tus aplicaciones no hablan directamente con el disco o la red — le piden al kernel que lo haga mediante **system calls** (syscalls).

```
┌──────────────────────────────────┐
│         Tus aplicaciones         │  Espacio de Usuario (userspace)
│    (Go, Python, bash, nginx)     │
├──────────────────────────────────┤
│       Bibliotecas (glibc)        │  Traducen funciones a syscalls
├──────────────────────────────────┤
│     System Calls (syscalls)      │  La frontera: open(), read(), write(), fork()
├──────────────────────────────────┤
│           KERNEL                 │  Espacio de Kernel
│   Scheduler | Memory | VFS | Net |
├──────────────────────────────────┤
│          HARDWARE                │  CPU, RAM, Disco, NIC
└──────────────────────────────────┘
```

**Conceptos clave**:

- **Syscall**: Cuando tu programa hace `open("/etc/hosts")`, eso se traduce en un syscall `open()` al kernel. El kernel verifica permisos, busca el archivo en el filesystem, y devuelve un file descriptor (un número entero).
- **File descriptor (fd)**: Todo en Linux es un archivo (o se comporta como uno). Cuando abrís un archivo, un socket, o un pipe, el kernel te da un fd. Los 3 estándar son: `0` (stdin), `1` (stdout), `2` (stderr).
- **Userspace vs Kernel space**: Tu código corre en userspace con acceso restringido. Solo el kernel puede acceder al hardware directamente. Esto es por seguridad y estabilidad.

**Ejercicio verificable**:

```bash
# Ver las syscalls que hace un comando simple
strace -c ls /tmp
# Verás un resumen de cuántas veces se llamó a cada syscall
# Preguntas: ¿Qué hace openat()? ¿Y getdents64()? ¿Y write()?

# Verificar:
strace ls /tmp 2>&1 | grep "^open\|^read\|^write" | head -20
# Deberías ver las llamadas reales al kernel
```

### 1.2 Procesos

Un **proceso** es un programa en ejecución. Cada proceso tiene:

- **PID**: Número único que lo identifica
- **PPID**: PID del proceso padre (quien lo creó)
- **UID/GID**: Usuario y grupo bajo el que corre
- **Estado**: Running, Sleeping, Stopped, Zombie
- **File descriptors**: Archivos, sockets, pipes abiertos
- **Espacio de memoria**: Propio, aislado de otros procesos

**Cómo nace un proceso**:

```
1. El proceso padre llama a fork() → se CLONA a sí mismo
2. El hijo resultante llama a exec() → REEMPLAZA su código con el nuevo programa
3. El padre puede wait() → esperar a que el hijo termine
```

Cuando hacés `ls` en bash:

1. Bash (PID 1234) llama a `fork()` → se crea un clon (PID 5678)
2. El clon (5678) llama a `exec("/usr/bin/ls")` → se convierte en `ls`
3. `ls` hace su trabajo y termina (exit)
4. Bash recibe la señal de que el hijo terminó

**Proceso zombi**: Cuando un hijo termina pero el padre no llamó a `wait()`. El proceso está muerto pero su entrada sigue en la tabla de procesos. No consume recursos pero indica un bug en el padre.

**Proceso huérfano**: Cuando el padre muere antes que el hijo. El sistema lo reasigna a PID 1 (init/systemd).

**Señales** (signals): Formas de comunicarse con un proceso:
| Señal | Número | Significado |
|-------|--------|-------------|
| SIGHUP | 1 | "Tu terminal se cerró" (se usa para reload de config) |
| SIGINT | 2 | Ctrl+C — "Pará por favor" |
| SIGKILL | 9 | "Morí ya" — NO se puede interceptar |
| SIGTERM | 15 | "Terminá limpiamente" — se puede interceptar |
| SIGSTOP | 19 | Pausar el proceso (Ctrl+Z) |
| SIGCONT | 18 | Reanudar un proceso pausado |

**Ejercicios verificables**:

```bash
# 1. Ver el árbol de procesos
pstree -p | head -30
# Pregunta: ¿Cuál es el PID 1? ¿De quién son hijos todos los demás?

# 2. Crear un proceso en background y manipularlo
sleep 300 &
echo "PID del sleep: $!"
ps aux | grep "sleep 300"
# Enviále SIGSTOP (pausar):
kill -STOP $!
ps aux | grep "sleep 300"  # Verás estado "T" (stopped)
# Reanudá con SIGCONT:
kill -CONT $!
# Matalo limpiamente:
kill -TERM $!

# 3. Crear un zombie (para entender qué es)
# Creá este script:
cat << 'EOF' > /tmp/zombie_demo.sh
#!/bin/bash
# El hijo muere pero el padre no hace wait()
bash -c "exit 0" &
sleep 60  # El padre duerme sin esperar al hijo
EOF
chmod +x /tmp/zombie_demo.sh
/tmp/zombie_demo.sh &
sleep 1
ps aux | grep -E "Z|defunct"
# Si ves un proceso con estado "Z" o "[defunct]", es un zombie
kill %1  # Matá el demo

# 4. Ver qué archivos tiene abiertos un proceso
# Abrí otra terminal y ejecutá:
lsof -p $$
# $$ es el PID de tu shell actual
# Verás stdin(0), stdout(1), stderr(2) y otros fd

# 5. Ver el directorio /proc de un proceso
ls /proc/$$/
cat /proc/$$/status | head -20
# /proc es un filesystem virtual que expone info del kernel sobre cada proceso
```

### 1.3 El Filesystem (Sistema de Archivos)

Linux tiene una **única jerarquía de directorios** que empieza en `/` (root). Todo se monta ahí, incluso otros discos, USBs, o filesystems de red.

**Estructura estándar (Filesystem Hierarchy Standard - FHS)**:

```
/
├── bin/      → Binarios esenciales (ls, cp, cat) [hoy symlink a /usr/bin]
├── boot/     → Kernel y bootloader (grub)
├── dev/      → Dispositivos como archivos (/dev/sda = disco, /dev/null = agujero negro)
├── etc/      → TODA la configuración del sistema
│   ├── passwd        → Lista de usuarios
│   ├── shadow        → Contraseñas hasheadas
│   ├── fstab         → Qué discos montar al arrancar
│   ├── hosts         → Resolución DNS local
│   ├── network/      → Configuración de red
│   └── systemd/      → Configuración de servicios
├── home/     → Directorios de usuarios (/home/Rolando)
├── lib/      → Bibliotecas compartidas (.so)
├── mnt/      → Punto de montaje temporal
├── opt/      → Software de terceros (instalaciones manuales)
├── proc/     → Filesystem virtual: info de procesos y kernel EN TIEMPO REAL
├── root/     → Home del usuario root
├── run/      → Datos de runtime (PIDs, sockets)
├── sbin/     → Binarios de administración (iptables, fdisk)
├── sys/      → Filesystem virtual: info del hardware
├── tmp/      → Archivos temporales (se borran al reiniciar)
├── usr/      → Programas y bibliotecas de usuario
│   ├── bin/          → Binarios de usuario
│   ├── lib/          → Bibliotecas
│   ├── local/        → Software compilado por el admin
│   └── share/        → Datos compartidos (man pages, docs)
└── var/      → Datos variables
    ├── log/          → TODOS los logs del sistema
    ├── lib/          → Datos persistentes de servicios
    └── tmp/          → Temporal persistente
```

**Tipos de filesystem**:
| Tipo | Uso | Características |
|------|-----|-----------------|
| ext4 | El estándar de Linux | Journaling, estable, hasta 1 exabyte |
| XFS | Servidores de alto rendimiento | Excelente para archivos grandes |
| btrfs | Moderno, con snapshots | Copy-on-write, compresión, subvolúmenes |
| tmpfs | RAM como filesystem | Ultra rápido, se borra al reiniciar (/tmp, /run) |
| procfs | Info del kernel | Virtual, se lee de la RAM no del disco (/proc) |
| sysfs | Info del hardware | Virtual (/sys) |

**Permisos**:

```
-rwxr-xr-- 1 rolando devops 4096 Mar 23 2026 script.sh
│├─┤├─┤├─┤   │        │
│ │  │  │     │        └── Grupo propietario
│ │  │  │     └── Usuario propietario
│ │  │  └── Otros: r-- (solo lectura)
│ │  └── Grupo: r-x (lectura + ejecución)
│ └── Owner: rwx (lectura + escritura + ejecución)
└── Tipo: - (archivo), d (directorio), l (symlink)
```

Permisos en octal:

- `r` = 4, `w` = 2, `x` = 1
- `rwxr-xr--` = 754
- `chmod 754 archivo` = owner todo, grupo lee+ejecuta, otros solo leen

**Permisos especiales**:

- **SUID** (4xxx): El programa se ejecuta con los permisos del OWNER, no del usuario que lo corre. Ejemplo: `passwd` tiene SUID porque necesita escribir en `/etc/shadow` (que es de root).
- **SGID** (2xxx): En directorios, los archivos nuevos heredan el grupo del directorio.
- **Sticky bit** (1xxx): En directorios, solo el propietario puede borrar sus archivos. Ejemplo: `/tmp`.

**Ejercicios verificables**:

```bash
# 1. Explorar /proc
cat /proc/cpuinfo | head -20      # Info de tu CPU
cat /proc/meminfo | head -10      # Info de memoria
cat /proc/version                 # Versión del kernel
cat /proc/loadavg                 # Carga del sistema
# Pregunta: ¿/proc/cpuinfo existe en el disco? ¿Ocupa espacio real?
# Respuesta: No. Es generado por el kernel en tiempo real.

# 2. Permisos
mkdir -p /tmp/permisos_lab
touch /tmp/permisos_lab/secreto.txt
chmod 600 /tmp/permisos_lab/secreto.txt
ls -la /tmp/permisos_lab/secreto.txt
# Solo tu usuario puede leer/escribir. Verificá:
stat /tmp/permisos_lab/secreto.txt | grep Access

# 3. Encontrar archivos SUID en el sistema (importante para seguridad)
find / -perm -4000 -type f 2>/dev/null | head -10
# Cada uno de estos se ejecuta como root. Un atacante busca estos archivos.

# 4. Entender inodes
ls -i /etc/hosts
# El número a la izquierda es el inode: el identificador REAL del archivo.
# El nombre es solo un "link" al inode.
stat /etc/hosts

# 5. Links duros vs simbólicos
echo "hola" > /tmp/original.txt
ln /tmp/original.txt /tmp/hardlink.txt      # Link duro (mismo inode)
ln -s /tmp/original.txt /tmp/symlink.txt    # Link simbólico (puntero)
ls -li /tmp/original.txt /tmp/hardlink.txt /tmp/symlink.txt
# Notá: original y hardlink tienen EL MISMO inode. symlink tiene otro.
# Si borrás original:
rm /tmp/original.txt
cat /tmp/hardlink.txt    # FUNCIONA (mismo inode, los datos siguen)
cat /tmp/symlink.txt     # FALLA (apuntaba al nombre, que ya no existe)
```

### 1.4 Usuarios, Grupos y su Modelo de Seguridad

Linux es un sistema **multiusuario**. Cada proceso corre bajo un usuario y grupo.

```bash
# Archivos clave:
/etc/passwd   → Lista de usuarios (nombre:x:UID:GID:info:home:shell)
/etc/shadow   → Contraseñas hasheadas (solo root puede leer)
/etc/group    → Lista de grupos (nombre:x:GID:miembros)
```

**UID especiales**:

- `0` → root (todopoderoso)
- `1-999` → Usuarios de sistema (servicios como nginx, postgres)
- `1000+` → Usuarios humanos

**¿Por qué los servicios tienen su propio usuario?** → **Principio de mínimo privilegio**: Si nginx tiene un bug que permite ejecución de código, el atacante solo tiene los permisos del usuario `nginx`, no de root. No puede leer `/etc/shadow`, no puede instalar software, no puede tocar otros servicios.

**Ejercicios verificables**:

```bash
# 1. Examinar tu usuario
id
# Muestra tu UID, GID y grupos secundarios

# 2. Ver usuarios del sistema
awk -F: '$3 < 1000 {print $1, $3}' /etc/passwd
# Todos estos son usuarios de servicio

# 3. Crear un usuario de práctica
sudo useradd -m -s /bin/bash practicante
sudo passwd practicante
su - practicante
whoami
pwd
exit

# 4. Entender sudo
sudo cat /etc/sudoers
# La línea "%sudo ALL=(ALL:ALL) ALL" significa:
# El grupo sudo puede ejecutar CUALQUIER comando como CUALQUIER usuario en CUALQUIER host

# 5. El peligro de root
# NUNCA trabajés como root directamente. Usá sudo.
# Verificá que tu usuario esté en el grupo sudo:
groups $USER | grep -o sudo
```

### 1.5 Bash Scripting Fundamental

Como Platform Engineer, vas a escribir scripts de bash constantemente — para automatización, health checks, deploys, y troubleshooting.

**Variables**:

```bash
# Asignación (SIN espacios alrededor del =)
nombre="Rolando"
edad=26

# Uso (con $)
echo "Hola $nombre, tenés $edad años"

# Variables especiales:
$0    # Nombre del script
$1    # Primer argumento
$#    # Número de argumentos
$?    # Exit code del último comando (0 = éxito)
$$    # PID del script actual
$!    # PID del último proceso en background
"$@"  # Todos los argumentos (preserva espacios)
```

**Condicionales**:

```bash
#!/bin/bash
# Verificar si un servicio está corriendo

servicio="sshd"

if systemctl is-active --quiet "$servicio"; then
    echo "✓ $servicio está corriendo"
else
    echo "✗ $servicio está detenido"
    echo "Intentando iniciar..."
    sudo systemctl start "$servicio"
fi

# Comparaciones de archivos:
# -f archivo    → ¿Existe y es un archivo regular?
# -d directorio → ¿Existe y es un directorio?
# -r archivo    → ¿Se puede leer?
# -w archivo    → ¿Se puede escribir?
# -x archivo    → ¿Se puede ejecutar?
# -s archivo    → ¿Existe y tiene tamaño > 0?

if [[ -f /etc/hosts ]]; then
    echo "/etc/hosts existe"
fi

# Comparaciones de strings:
# == , != , -z (vacío), -n (no vacío)

# Comparaciones numéricas:
# -eq, -ne, -lt, -le, -gt, -ge
```

**Loops**:

```bash
#!/bin/bash
# Iterar sobre archivos de log
for logfile in /var/log/*.log; do
    tamanio=$(du -sh "$logfile" | cut -f1)
    echo "$logfile: $tamanio"
done

# While: leer línea por línea
while IFS= read -r linea; do
    echo "Procesando: $linea"
done < /etc/hosts

# Until: repetir hasta que se cumpla
intentos=0
until ping -c 1 -W 1 8.8.8.8 &>/dev/null; do
    ((intentos++))
    echo "Intento $intentos: sin conexión..."
    sleep 2
done
echo "Conectado después de $intentos intentos"
```

**Funciones**:

```bash
#!/bin/bash

log_info() {
    echo "[INFO $(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log_error() {
    echo "[ERROR $(date '+%Y-%m-%d %H:%M:%S')] $1" >&2
}

verificar_disco() {
    local umbral="${1:-80}"  # Default 80%
    local uso
    uso=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

    if [[ $uso -ge $umbral ]]; then
        log_error "Disco al ${uso}% — supera el umbral de ${umbral}%"
        return 1
    else
        log_info "Disco al ${uso}% — dentro del umbral"
        return 0
    fi
}

# Llamar la función
verificar_disco 90
```

**Ejercicio integrador del mes 1 — Script de salud del sistema**:

```bash
#!/bin/bash
# health_check.sh — Tu primer script útil de verdad
# Verificá que podés escribirlo SIN copiar y pegar

set -euo pipefail  # Modo estricto: fallar ante errores

UMBRAL_DISCO=80
UMBRAL_MEMORIA=90
UMBRAL_CPU=80

echo "=== Health Check del Sistema ==="
echo "Fecha: $(date)"
echo "Hostname: $(hostname)"
echo "Uptime: $(uptime -p)"
echo ""

# 1. Verificar uso de disco
echo "--- Disco ---"
while IFS= read -r linea; do
    uso=$(echo "$linea" | awk '{print $5}' | tr -d '%')
    montaje=$(echo "$linea" | awk '{print $6}')
    if [[ $uso -ge $UMBRAL_DISCO ]]; then
        echo "⚠ ALERTA: $montaje al ${uso}%"
    else
        echo "✓ $montaje: ${uso}%"
    fi
done < <(df -h | awk 'NR>1 && /^\/dev/')

# 2. Verificar memoria
echo ""
echo "--- Memoria ---"
mem_total=$(free -m | awk '/^Mem:/ {print $2}')
mem_usada=$(free -m | awk '/^Mem:/ {print $3}')
mem_pct=$((mem_usada * 100 / mem_total))
if [[ $mem_pct -ge $UMBRAL_MEMORIA ]]; then
    echo "⚠ ALERTA: Memoria al ${mem_pct}% (${mem_usada}MB / ${mem_total}MB)"
else
    echo "✓ Memoria: ${mem_pct}% (${mem_usada}MB / ${mem_total}MB)"
fi

# 3. Top 5 procesos por CPU
echo ""
echo "--- Top 5 Procesos (CPU) ---"
ps aux --sort=-%cpu | awk 'NR<=6 {printf "%-10s %5s%% %s\n", $1, $3, $11}'

# 4. Top 5 procesos por memoria
echo ""
echo "--- Top 5 Procesos (MEM) ---"
ps aux --sort=-%mem | awk 'NR<=6 {printf "%-10s %5s%% %s\n", $1, $4, $11}'

echo ""
echo "=== Fin del Health Check ==="
```

**Cómo verificar que aprendiste**: Escribí el script de arriba de cero, sin mirarlo. Si podés, dominás lo básico de bash.

---

## Mes 2: Redes TCP/IP

### 2.1 El Modelo de Capas

Todo lo que viaja por la red pasa por capas. Cada capa agrega su encabezado (**header**) al paquete:

```
DATOS DE TU APP
    ↓ Capa 7 (Aplicación: HTTP, DNS, SSH)
[HTTP Header][DATOS]
    ↓ Capa 4 (Transporte: TCP o UDP)
[TCP Header][HTTP Header][DATOS]
    ↓ Capa 3 (Red: IP)
[IP Header][TCP Header][HTTP Header][DATOS]
    ↓ Capa 2 (Enlace: Ethernet/WiFi)
[ETH Header][IP Header][TCP Header][HTTP Header][DATOS][ETH Trailer]
    ↓ Capa 1 (Física: cable, ondas)
    Señales eléctricas / luz / radio
```

Cuando el receptor lo recibe, va quitando capas (como pelar una cebolla) hasta llegar a los datos originales.

### 2.2 Capa 3: IP (Internet Protocol)

**Dirección IP**: Identificador numérico de un dispositivo en la red.

**IPv4**: 4 bytes (32 bits) → `192.168.1.100`

- Cada número va de 0 a 255 (un byte = 8 bits)
- Máximo teórico: ~4.3 mil millones de direcciones (se acabaron en 2011)

**Subredes y máscara**:

```
IP:      192.168.1.100
Máscara: 255.255.255.0  (/24)

Esto significa:
- Los primeros 24 bits son la RED:     192.168.1.___
- Los últimos 8 bits son el HOST:      ___.___.___.100

Todos los dispositivos con 192.168.1.X están en la MISMA red local.
Para hablar con 192.168.2.X necesitás un ROUTER (gateway).
```

**Notación CIDR**: `/24` significa "los primeros 24 bits son la red"

- `/24` = 256 IPs (254 usables) → Red típica de casa/oficina
- `/16` = 65,536 IPs → Red corporativa
- `/32` = 1 sola IP → Un host específico
- `/0` = Todas las IPs → "Cualquier dirección" (default route)

**IPs privadas** (no ruteables en Internet):
| Rango | CIDR | Cantidad |
|-------|------|----------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | ~16 millones |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | ~1 millón |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | ~65 mil |

**NAT (Network Address Translation)**: Tu router traduce tu IP privada (192.168.1.100) a su IP pública cuando salís a Internet. El servidor de destino solo ve la IP pública del router.

**Tabla de ruteo**: El kernel de Linux decide para cada paquete: "¿a dónde lo mando?"

```bash
ip route show
# Verás algo como:
# default via 192.168.1.1 dev eth0    → "Todo lo que no conozca, mandalo al router"
# 192.168.1.0/24 dev eth0             → "Para esta red, mandalo directo por eth0"
```

### 2.3 Capa 4: TCP y UDP

**TCP (Transmission Control Protocol)**: Confiable, ordenado, con conexión.

- **Handshake de 3 vías** para establecer conexión:
  ```
  Cliente → SYN → Servidor       "¿Puedo conectarme?"
  Cliente ← SYN-ACK ← Servidor   "Sí, dale"
  Cliente → ACK → Servidor        "Ok, empezamos"
  ```
- Cada paquete tiene un **número de secuencia** → garantiza orden
- Cada paquete recibido se **confirma (ACK)** → si no llega ACK, se reenvía
- **Control de flujo**: el receptor dice "mandame más lento" si no da abasto
- **Usado por**: HTTP, SSH, SMTP, bases de datos — todo lo que necesita confiabilidad

**UDP (User Datagram Protocol)**: Rápido, sin conexión, sin garantías.

- No hay handshake — mandás el paquete y listo
- No hay reenvío si se pierde
- No hay orden garantizado
- **Usado por**: DNS (consultas rápidas), video streaming, gaming, VoIP

**Puertos**: Un número (0-65535) que identifica qué servicio recibe el paquete.
| Puerto | Servicio | Protocolo |
|--------|----------|-----------|
| 22 | SSH | TCP |
| 53 | DNS | TCP/UDP |
| 80 | HTTP | TCP |
| 443 | HTTPS | TCP |
| 5432 | PostgreSQL | TCP |
| 8081 | Tu API GraphQL | TCP |

La combinación `IP:Puerto` identifica un servicio específico en una máquina. Tu API corre en `192.168.x.x:8081`.

### 2.4 DNS (Domain Name System)

DNS traduce nombres (`google.com`) a IPs (`142.250.184.206`). Es el "directorio telefónico de Internet".

**Cómo funciona una consulta DNS**:

```
1. Escribís "google.com" en el navegador
2. Tu PC mira /etc/hosts → ¿está ahí? Si sí, usa esa IP
3. Si no, pregunta al DNS resolver configurado (/etc/resolv.conf)
4. El resolver pregunta a los Root servers → "¿Quién sabe de .com?"
5. Los Root servers dicen → "Preguntale a los servidores de .com"
6. Los servidores de .com dicen → "google.com está en ns1.google.com"
7. ns1.google.com responde → "google.com = 142.250.184.206"
8. El resolver cachea la respuesta y te la devuelve
```

**Tipos de registros DNS**:
| Tipo | Propósito | Ejemplo |
|------|-----------|---------|
| A | Nombre → IPv4 | google.com → 142.250.184.206 |
| AAAA | Nombre → IPv6 | google.com → 2607:f8b0:4004:800::200e |
| CNAME | Alias → Otro nombre | www.google.com → google.com |
| MX | Servidor de mail | google.com → smtp.google.com |
| NS | Servidor DNS autoritativo | google.com → ns1.google.com |
| TXT | Texto libre (verificación, SPF) | google.com → "v=spf1 ..." |
| PTR | IP → Nombre (reverso) | 142.250.184.206 → google.com |

**Ejercicios verificables**:

```bash
# 1. Resolver DNS manualmente
dig google.com
# Mirá la sección ANSWER: ahí está la IP
# Mirá "Query time": cuánto tardó en milisegundos
# Ejecutá de nuevo: el tiempo baja (cacheado)

# 2. Trazar la cadena de DNS
dig +trace google.com
# Verás paso a paso: root → .com → google.com

# 3. Consultar diferentes tipos de registro
dig google.com MX      # Servidores de mail
dig google.com NS      # Servidores DNS
dig google.com TXT     # Registros TXT

# 4. Ver tu configuración DNS
cat /etc/resolv.conf
# "nameserver 8.8.8.8" significa que usás el DNS de Google

# 5. Resolución local
cat /etc/hosts
# Esto tiene prioridad sobre DNS. Podés "mentir" al sistema:
# sudo echo "127.0.0.1 miapp.local" >> /etc/hosts
# Ahora "miapp.local" apunta a tu máquina
```

### 2.5 HTTP (HyperText Transfer Protocol)

Tu API GraphQL habla HTTP. Necesitás entenderlo en profundidad.

**Anatomía de un request HTTP**:

```
POST /query HTTP/1.1              ← Método, path, versión
Host: localhost:8081              ← Headers
Content-Type: application/json
Authorization: Bearer xyz123
                                   ← Línea vacía separa headers de body
{"query": "{ products { id name } }"}  ← Body
```

**Métodos HTTP**:
| Método | Propósito | Idempotente | Body |
|--------|-----------|-------------|------|
| GET | Leer recurso | Sí | No (usualmente) |
| POST | Crear recurso | No | Sí |
| PUT | Reemplazar recurso completo | Sí | Sí |
| PATCH | Modificar parcialmente | No | Sí |
| DELETE | Eliminar recurso | Sí | Opcional |

**Idempotente** = Si lo ejecutás 2 veces, el resultado es el mismo. GET el mismo recurso 2 veces da lo mismo. DELETE el mismo recurso 2 veces: la primera borra, la segunda da 404, pero el estado final es el mismo.

**Códigos de respuesta**:
| Rango | Significado | Ejemplos |
|-------|-------------|----------|
| 2xx | Éxito | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirección | 301 Moved Permanently, 304 Not Modified |
| 4xx | Error del cliente | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Error del servidor | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

**HTTPS**: HTTP + TLS (Transport Layer Security). El tráfico va encriptado.

- El servidor presenta un **certificado** que prueba su identidad
- Se negocia una **clave de sesión** para encriptar la comunicación
- Toda la data viaja encriptada — nadie entre tu PC y el servidor puede leerla

**Ejercicios verificables**:

```bash
# 1. Hacer un request HTTP y ver todo el detalle
curl -v http://httpbin.org/get
# -v muestra headers del request Y de la respuesta

# 2. POST con body JSON (como tu API GraphQL)
curl -X POST http://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Rolando"}'

# 3. Ver solo los headers de respuesta
curl -I https://google.com
# Mirá: HTTP/2 301 (redirección a www.google.com)

# 4. Seguir redirecciones
curl -L -v https://google.com 2>&1 | grep "< HTTP"
# Verás el 301 y después el 200

# 5. Probar TU API (si está corriendo)
curl -X POST http://localhost:8081/query \
  -H "Content-Type: application/json" \
  -d '{"query": "{ products { id name price } }"}'
```

### 2.6 Firewalls y iptables/nftables

Un **firewall** filtra tráfico de red basándose en reglas: "permitir SSH desde esta IP", "bloquear todo lo que venga de Internet al puerto 5432".

**iptables** (el clásico) / **nftables** (el sucesor):

```bash
# Ver reglas actuales
sudo iptables -L -n -v

# Ejemplo: Permitir SSH, HTTP y bloquear el resto
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # Respuestas
sudo iptables -A INPUT -j DROP                          # Todo lo demás: bloqueado
```

**Cadenas** (chains):

- **INPUT**: Tráfico que LLEGA a este host
- **OUTPUT**: Tráfico que SALE de este host
- **FORWARD**: Tráfico que PASA POR este host (es un router)

**Ejercicio verificable**:

```bash
# Ver las reglas de firewall actuales
sudo iptables -L -n --line-numbers

# Ver puertos abiertos en tu máquina
ss -tlnp
# t=TCP, l=listening, n=numérico, p=proceso
# Cada línea es un servicio esperando conexiones
```

---

## Mes 3: Herramientas de Red y Troubleshooting

### 3.1 El Toolkit del Administrador de Redes

```bash
# 1. ip — Configuración de interfaces de red (reemplazo de ifconfig)
ip addr show              # Ver IPs de todas las interfaces
ip link show              # Ver estado de interfaces
ip route show             # Ver tabla de ruteo

# 2. ss — Estado de sockets/conexiones (reemplazo de netstat)
ss -tlnp                  # Puertos TCP en escucha
ss -ulnp                  # Puertos UDP en escucha
ss -tn state established  # Conexiones TCP activas

# 3. ping — Verificar conectividad ICMP
ping -c 4 8.8.8.8        # 4 paquetes al DNS de Google
# Si falla: problema de red/firewall
# Si funciona: la red base está bien

# 4. traceroute — Ver la ruta de paquetes
traceroute 8.8.8.8
# Muestra cada router por el que pasa el paquete
# Útil para encontrar dónde se pierde la conexión

# 5. dig/nslookup — Consultas DNS
dig example.com           # Consulta DNS completa
dig +short example.com    # Solo la IP

# 6. curl — Cliente HTTP versátil
curl -v https://api.github.com  # Ver todo el intercambio HTTP+TLS

# 7. tcpdump — Capturar paquetes (wireshark de terminal)
sudo tcpdump -i any -n port 80     # Capturar tráfico HTTP
sudo tcpdump -i any -n host 8.8.8.8  # Tráfico hacia/desde 8.8.8.8

# 8. nmap — Escanear puertos abiertos
nmap -sT localhost        # ¿Qué puertos TCP están abiertos aquí?
nmap -sT 192.168.1.1      # ¿Qué puertos tiene expuestos el router?
```

### 3.2 Metodología de Troubleshooting de Red

Cuando algo en la red no funciona, seguí esta secuencia:

```
1. ¿Tengo IP?
   → ip addr show

2. ¿Puedo llegar al gateway?
   → ping <IP del gateway>

3. ¿Puedo llegar a Internet?
   → ping 8.8.8.8

4. ¿DNS funciona?
   → dig google.com

5. ¿El puerto específico está abierto?
   → ss -tlnp | grep <puerto>
   → curl -v http://<host>:<puerto>

6. ¿Hay firewall bloqueando?
   → sudo iptables -L -n

7. ¿Llega el tráfico?
   → sudo tcpdump -i any port <puerto>
```

### 3.3 Ejercicio Integrador — Troubleshooting Real

```bash
# Simulamos un problema: "la API no responde en el puerto 8081"
# Investigá paso a paso:

# 1. ¿El proceso está corriendo?
ps aux | grep "gestion_productos"
# Si no está → el binario no se ejecutó o crasheó

# 2. ¿Está escuchando en el puerto?
ss -tlnp | grep 8081
# Si no está → el proceso crasheó al iniciar, revisá logs

# 3. ¿Puedo conectarme localmente?
curl -v http://localhost:8081/
# Si falla → el servicio no responde, revisá el proceso

# 4. ¿Puedo conectarme desde otra máquina?
# Desde otra PC: curl http://<IP>:8081/
# Si falla pero localhost funciona → firewall bloqueándolo

# 5. ¿El firewall lo bloquea?
sudo iptables -L -n | grep 8081

# 6. ¿Llega el tráfico?
sudo tcpdump -i any port 8081
# Si ves paquetes SYN pero no SYN-ACK → el servicio no responde
# Si no ves nada → el tráfico no llega (problema de red/firewall)
```

---

## Proyecto Integrador de Fase 1

### "Mi Propio Servidor Web desde Cero"

Usando solo lo aprendido en esta fase, hacé lo siguiente en una VM de práctica (o tu misma PC si te animás):

1. **Creá un usuario dedicado** para el servicio (sin acceso a sudo)
2. **Escribí un script bash** (`server_monitor.sh`) que cada 5 minutos:
   - Verifique uso de disco, RAM y CPU
   - Escriba los resultados en `/var/log/monitor/$(date +%Y-%m-%d).log`
   - Si algo supera un umbral, escriba una alerta en un archivo separado
3. **Configurá permisos** correctamente:
   - El script pertenece a root pero el usuario de servicio puede ejecutarlo (SGID)
   - Los logs solo los puede leer el grupo `monitoring`
4. **Verificá con la red** que tu API de productos funciona:
   - Mostrá con `ss` que está escuchando
   - Mostrá con `curl` que responde
   - Capturá un request con `tcpdump`
5. **Escribí reglas de firewall** que solo permitan SSH (22) y tu API (8081)

**Verificación**:

```bash
# Tu checklist debe dar todo ok:
id servicio_user                        # ✓ El usuario existe
ls -la /usr/local/bin/server_monitor.sh # ✓ Permisos correctos
cat /var/log/monitor/$(date +%Y-%m-%d).log  # ✓ Hay datos
ss -tlnp | grep -E "22|8081"           # ✓ Solo estos puertos
sudo iptables -L -n                    # ✓ Reglas correctas
curl http://localhost:8081/             # ✓ La API responde
```

---

## Recursos para esta Fase

### Libros (en orden de prioridad)

1. **"How Linux Works" de Brian Ward** (3era edición) — EL libro para entender Linux por dentro. Claro, práctico, sin presumir. _Compralo o buscá en Library Genesis._
2. **"The Linux Command Line" de William Shotts** — Gratis en linuxcommand.org. Excelente para bash.
3. **"TCP/IP Illustrated Vol. 1" de Kevin Fall** — La biblia de redes. Denso pero completo.
4. **"Computer Networking: A Top-Down Approach" de Kurose & Ross** — Más accesible que el anterior.

### Cursos y Labs Gratuitos

- **Linux Journey** (linuxjourney.com) — Interactivo, paso a paso
- **SadServers** (sadservers.com) — Troubleshooting en VMs reales. Excelente.
- **OverTheWire: Bandit** (overthewire.org/wargames/bandit) — Aprendé Linux jugando un CTF
- **Networking Fundamentals - Practical Networking** (YouTube) — Serie de videos clara y visual

### Documentación Oficial

- `man <comando>` — Las man pages son tu primera fuente. Ejemplo: `man ip`, `man ss`
- **Arch Wiki** (wiki.archlinux.org) — La mejor documentación de Linux del mundo, aplica a cualquier distro

### Práctica Diaria

- Cada vez que necesites hacer algo, intentá en la terminal antes de buscar en Google
- Usá `man` antes de Stack Overflow
- Explorá `/proc`, `/sys`, `/etc` regularmente — familiarizate con tu sistema

---

## Checkpoint: ¿Estoy listo para la Fase 2?

Podés responder estas preguntas sin googlear:

- [ ] ¿Qué es un syscall y para qué sirve?
- [ ] ¿Qué es un file descriptor?
- [ ] ¿Cuál es la diferencia entre SIGTERM y SIGKILL?
- [ ] ¿Qué hace `chmod 750`?
- [ ] ¿Qué es un inode?
- [ ] ¿Diferencia entre hard link y symbolic link?
- [ ] ¿Qué rango de IPs es privado?
- [ ] ¿Diferencia entre TCP y UDP?
- [ ] ¿Qué es un handshake de 3 vías?
- [ ] ¿Cómo funciona una consulta DNS?
- [ ] ¿Qué es NAT?
- [ ] ¿Qué hace `ss -tlnp`?
- [ ] ¿Cómo diagnosticás por qué un servicio no responde en la red?

Si respondiste 10+ de 13: avanzá a la Fase 2.
