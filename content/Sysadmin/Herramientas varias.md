[[0. General Tips]]

# Herramientas Varias

## 1. Vagrant — Máquinas Virtuales Descartables

Ideal para practicar y romper cosas sin consecuencias.

```bash
# Iniciar un proyecto con Debian
vagrant init debian/bookworm64

# Levantar la VM
vagrant up

# Conectarse por SSH
vagrant ssh

# Guardar un snapshot (punto de restauración)
vagrant snapshot save "antes-de-romper-todo"

# Restaurar el snapshot
vagrant snapshot restore "antes-de-romper-todo"

# Apagar la VM
vagrant halt

# Destruir la VM completamente
vagrant destroy
```

---

## 2. tmux — Multiplexor de Terminal

Permite tener múltiples terminales en una sola ventana y mantener sesiones activas aunque cierres SSH.

```bash
# Iniciar una nueva sesión
tmux

# Iniciar sesión con nombre
tmux new -s mi-sesion

# Desconectarse sin cerrar (detach)
Ctrl+b luego d

# Ver sesiones activas
tmux ls

# Reconectarse a una sesión
tmux attach -t mi-sesion

# Dividir pantalla horizontalmente
Ctrl+b luego "

# Dividir pantalla verticalmente
Ctrl+b luego %

# Moverse entre paneles
Ctrl+b luego flechas

# Cerrar sesión
exit
```

> **Caso de uso:** Conectarte por SSH a un servidor, iniciar tmux, ejecutar un proceso largo. Si se corta la conexión SSH, el proceso sigue corriendo en tmux. Te reconectás y todo sigue ahí.

---

## 3. rsync — Sincronización y Backups

Más inteligente que `cp` o `scp`: solo copia lo que cambió.

```bash
# Sincronizar carpeta local
rsync -avh /origen/ /destino/

# Sincronizar hacia un servidor remoto
rsync -avh /carpeta/local/ usuario@IP:/carpeta/remota/

# Sincronizar desde un servidor remoto
rsync -avh usuario@IP:/carpeta/remota/ /carpeta/local/

# Con barra de progreso y compresión
rsync -avhz --progress /origen/ /destino/

# Borrar en destino los archivos que ya no existen en origen
rsync -avh --delete /origen/ /destino/
```

| Flag        | Significado                                           |
| ----------- | ----------------------------------------------------- |
| `-a`        | Archive (preserva permisos, fechas, links, etc.)      |
| `-v`        | Verbose (muestra qué está haciendo)                   |
| `-h`        | Tamaños legibles para humanos                         |
| `-z`        | Comprimir durante la transferencia                    |
| `--delete`  | Eliminar archivos en destino que no existen en origen |
| `--dry-run` | Simular sin hacer nada (para verificar antes)         |

---

## 4. wget y curl — Descargar desde la Terminal

```bash
# wget: descargar un archivo
wget https://ejemplo.com/archivo.tar.gz

# wget: descargar en segundo plano (se puede cerrar la terminal)
wget -b https://ejemplo.com/archivo.iso

# curl: descargar un archivo
curl -O https://ejemplo.com/archivo.tar.gz

# curl: ver contenido de una URL (sin descargar)
curl https://ejemplo.com/api/salud

# curl: ver headers HTTP
curl -I https://ejemplo.com

# curl: enviar datos POST
curl -X POST -H "Content-Type: application/json" -d '{"key": "value"}' https://api.ejemplo.com/datos
```

---

## 5. Otras Herramientas Útiles

```bash
# Ver el espacio que ocupa cada carpeta (interactivo)
ncdu /

# Monitorear tráfico de red en tiempo real
sudo iftop

# Monitorear I/O de disco
sudo iotop

# Verificar puertos abiertos rápidamente
nmap -sT localhost

# Generar contraseñas aleatorias
openssl rand -base64 32

# Ver información del sistema
hostnamectl
lsb_release -a     # Distribución
uname -a            # Kernel
```
