[[0. General Tips]]

# Gestión de Discos y Almacenamiento

## 1. Ver Discos y Particiones

```bash
# Listar todos los discos y particiones (vista de árbol)
lsblk

# Información detallada de particiones
sudo fdisk -l

# Ver identificadores UUID de las particiones
blkid

# Ver puntos de montaje activos
mount | column -t

# Ver espacio usado por partición
df -hT
```

**Nomenclatura de discos en Linux:**

- `sda`, `sdb`, `sdc` → Discos SATA/SAS/USB
- `nvme0n1`, `nvme1n1` → Discos NVMe
- `vda`, `vdb` → Discos virtuales (KVM/QEMU)
- `sda1`, `sda2` → Particiones del disco `sda`

---

## 2. Particionar Discos

### Con `fdisk` (discos < 2TB, tabla MBR)

```bash
sudo fdisk /dev/sdb
```

Comandos dentro de fdisk:

| Tecla | Acción                    |
| ----- | ------------------------- |
| `p`   | Ver particiones actuales  |
| `n`   | Crear nueva partición     |
| `d`   | Eliminar partición        |
| `t`   | Cambiar tipo de partición |
| `w`   | Escribir cambios y salir  |
| `q`   | Salir sin guardar         |

### Con `parted` (discos > 2TB, tabla GPT)

```bash
sudo parted /dev/sdb

# Dentro de parted:
mklabel gpt                           # Crear tabla GPT
mkpart primary ext4 0% 100%           # Crear partición
print                                 # Ver particiones
quit
```

---

## 3. Formatear Particiones (Crear Filesystem)

```bash
# Formatear como ext4 (el estándar en Linux)
sudo mkfs.ext4 /dev/sdb1

# Formatear como XFS (mejor para archivos grandes)
sudo mkfs.xfs /dev/sdb1

# Formatear swap
sudo mkswap /dev/sdb2
```

---

## 4. Montar y Desmontar

```bash
# Montar una partición
sudo mount /dev/sdb1 /mnt/datos

# Montar con opciones específicas
sudo mount -o ro /dev/sdb1 /mnt/datos       # Solo lectura
sudo mount -o noexec /dev/sdb1 /mnt/datos   # No permitir ejecutables

# Desmontar
sudo umount /mnt/datos

# Si dice "device is busy", ver quién lo usa
sudo lsof +f -- /mnt/datos
# O forzar desmontar (último recurso)
sudo umount -l /mnt/datos
```

### Montaje Permanente con `/etc/fstab`

Para que una partición se monte automáticamente al arrancar:

```bash
# 1. Obtener el UUID de la partición
blkid /dev/sdb1

# 2. Editar fstab
sudo vim /etc/fstab
```

Formato de `/etc/fstab`:

```
# <dispositivo>       <punto_montaje>  <tipo>  <opciones>       <dump> <pass>
UUID=xxxx-xxxx-xxxx   /mnt/datos       ext4    defaults         0      2
UUID=yyyy-yyyy-yyyy   none             swap    sw               0      0
```

> **Importante:** Siempre usar UUID en vez de `/dev/sdX` — los nombres de dispositivo pueden cambiar entre boots.

```bash
# 3. Probar sin reiniciar (monta todo lo que está en fstab)
sudo mount -a

# Si hay un error en fstab y no arranca, boot en recovery y arreglar el archivo
```

> **CUIDADO:** Un error en `/etc/fstab` puede hacer que el servidor no arranque. Siempre probar con `mount -a` antes de reiniciar.

---

## 5. Swap (Memoria de Intercambio)

```bash
# Ver swap actual
swapon --show
free -h

# Crear archivo swap de 2GB
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Hacerlo permanente (agregar a fstab)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Desactivar swap
sudo swapoff /swapfile

# Ajustar "swappiness" (cuánto usa swap vs RAM)
# Valor bajo (10) = prefiere RAM. Valor alto (60) = usa más swap
cat /proc/sys/vm/swappiness                    # Ver actual
sudo sysctl vm.swappiness=10                   # Cambiar temporalmente
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf  # Permanente
```

---

## 6. LVM (Logical Volume Manager)

LVM permite redimensionar particiones en caliente, crear snapshots y gestionar almacenamiento flexible.

### Conceptos

```
Disco Físico (/dev/sdb)
    └── PV (Physical Volume) — El disco preparado para LVM
        └── VG (Volume Group) — Un "pool" de almacenamiento
            └── LV (Logical Volume) — La "partición" lógica que se monta
```

### Crear un LVM desde cero

```bash
# 1. Crear Physical Volume
sudo pvcreate /dev/sdb

# 2. Crear Volume Group
sudo vgcreate datos-vg /dev/sdb

# 3. Crear Logical Volume (usar 80% del espacio)
sudo lvcreate -l 80%FREE -n app-lv datos-vg

# 4. Formatear
sudo mkfs.ext4 /dev/datos-vg/app-lv

# 5. Montar
sudo mkdir -p /mnt/app
sudo mount /dev/datos-vg/app-lv /mnt/app

# 6. Agregar a fstab para persistencia
echo '/dev/datos-vg/app-lv /mnt/app ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

### Operaciones Comunes

```bash
# Ver estado de PV, VG, LV
sudo pvs
sudo vgs
sudo lvs

# Información detallada
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay

# EXTENDER un Logical Volume (agregar espacio)
sudo lvextend -L +10G /dev/datos-vg/app-lv    # Agregar 10GB
sudo lvextend -l +100%FREE /dev/datos-vg/app-lv  # Usar todo el espacio libre

# IMPORTANTE: Después de extender, redimensionar el filesystem
sudo resize2fs /dev/datos-vg/app-lv            # Para ext4
sudo xfs_growfs /mnt/app                        # Para XFS

# Agregar un disco nuevo al VG existente
sudo pvcreate /dev/sdc
sudo vgextend datos-vg /dev/sdc
# Ahora el VG tiene más espacio y podés extender los LV
```

### Snapshots LVM

```bash
# Crear snapshot (útil antes de actualizaciones riesgosas)
sudo lvcreate -s -n app-snapshot -L 5G /dev/datos-vg/app-lv

# Restaurar desde snapshot (si algo salió mal)
sudo lvconvert --merge /dev/datos-vg/app-snapshot
# Requiere reiniciar si el LV está montado
```

---

## 7. RAID por Software (mdadm)

RAID combina múltiples discos para redundancia o rendimiento.

| Nivel   | Discos Mínimos | Tolerancia a Fallos  | Uso                                 |
| ------- | -------------- | -------------------- | ----------------------------------- |
| RAID 0  | 2              | Ninguna              | Solo velocidad (NO para producción) |
| RAID 1  | 2              | 1 disco puede fallar | Espejo, ideal para OS               |
| RAID 5  | 3              | 1 disco puede fallar | Balance velocidad/redundancia       |
| RAID 10 | 4              | 1 disco por espejo   | Mejor rendimiento + redundancia     |

```bash
# Crear RAID 1 (espejo) con 2 discos
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Ver estado del RAID
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Formatear y montar como cualquier partición
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /mnt/raid

# Guardar configuración
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

# Si un disco falla, reemplazar
sudo mdadm --manage /dev/md0 --remove /dev/sdc   # Quitar el que falló
sudo mdadm --manage /dev/md0 --add /dev/sdd      # Agregar el nuevo
# El RAID se reconstruye automáticamente
```

---

## 8. Verificar Salud de Discos

```bash
# Ver salud SMART del disco
sudo smartctl -a /dev/sda

# Test rápido
sudo smartctl -t short /dev/sda

# Verificar filesystem (SOLO con partición desmontada)
sudo fsck /dev/sda1

# Ver errores del disco en los logs
sudo dmesg | grep -i "error\|fault\|fail" | grep -i "sd\|nvme"
```
