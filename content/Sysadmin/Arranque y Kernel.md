[[0. General Tips]]

# Arranque del Sistema y Kernel

## 1. El Proceso de Boot (Secuencia Completa)

```
1. BIOS/UEFI
   └── Busca un dispositivo booteable (disco, USB, red)

2. Bootloader (GRUB)
   └── Presenta menú de kernels
   └── Carga el kernel y el initramfs en memoria

3. Kernel
   └── Detecta hardware
   └── Monta el initramfs (filesystem temporal en RAM)
   └── Monta el filesystem raíz real (/)

4. Systemd (PID 1)
   └── Lee /etc/fstab (monta particiones)
   └── Arranca servicios (targets)
   └── Llega al target final (multi-user o graphical)

5. Login
   └── getty muestra el prompt de login en TTY
   └── O el display manager muestra login gráfico
```

---

## 2. GRUB (Grand Unified Bootloader)

### Archivo de configuración

```bash
# Configuración principal (NO editar directamente)
/boot/grub/grub.cfg

# Editar ESTE archivo y luego regenerar
sudo vim /etc/default/grub
```

### Opciones comunes en `/etc/default/grub`

```bash
# Timeout del menú (segundos)
GRUB_TIMEOUT=5

# Kernel por defecto (0 = primer entrada)
GRUB_DEFAULT=0

# O usar el último kernel seleccionado
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true

# Parámetros del kernel
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# Para ver los mensajes de boot (útil en servidores)
GRUB_CMDLINE_LINUX_DEFAULT=""
```

### Aplicar cambios

```bash
# Debian/Ubuntu
sudo update-grub

# RHEL/Arch (equivalente)
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Reparar GRUB (si no arranca)

```bash
# Desde un Live USB:
# 1. Montar la partición raíz
sudo mount /dev/sda2 /mnt

# 2. Si tenés partición EFI separada
sudo mount /dev/sda1 /mnt/boot/efi

# 3. Montar filesystems especiales
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys

# 4. Entrar al sistema
sudo chroot /mnt

# 5. Reinstalar GRUB
grub-install /dev/sda          # Para BIOS/MBR
grub-install --target=x86_64-efi --efi-directory=/boot/efi   # Para UEFI
update-grub

# 6. Salir y reiniciar
exit
sudo umount -R /mnt
sudo reboot
```

---

## 3. Modo Rescue / Emergency

Si el sistema no arranca correctamente:

### Desde el menú de GRUB

1. En el menú de GRUB, presionar `e` para editar la entrada
2. Buscar la línea que empieza con `linux` o `linux16`
3. Agregar al final de la línea:
   - `single` o `1` → Modo single-user (con red)
   - `systemd.unit=rescue.target` → Modo rescue
   - `systemd.unit=emergency.target` → Modo emergency (mínimo)
   - `init=/bin/bash` → Shell directa (último recurso)
4. Presionar `Ctrl+X` para arrancar

### Modo Emergency (filesystem en solo lectura)

```bash
# Si entraste en emergency mode, el filesystem está en read-only
# Remontarlo como read-write:
mount -o remount,rw /

# Arreglar lo que haya que arreglar (fstab, etc.)
vim /etc/fstab

# Reiniciar
reboot
```

---

## 4. Kernel

### Información del Kernel

```bash
# Ver versión del kernel actual
uname -r

# Información completa
uname -a

# Ver todos los kernels instalados (Debian/Ubuntu)
dpkg --list | grep linux-image

# Ver todos los kernels instalados (RHEL/Fedora)
rpm -qa | grep kernel
```

### Módulos del Kernel

Los módulos son "drivers" que se cargan dinámicamente.

```bash
# Ver módulos cargados
lsmod

# Información de un módulo
modinfo nombre_modulo

# Cargar un módulo
sudo modprobe nombre_modulo

# Descargar un módulo
sudo modprobe -r nombre_modulo

# Cargar un módulo automáticamente al arrancar
echo "nombre_modulo" | sudo tee /etc/modules-load.d/nombre.conf

# Blacklist: impedir que un módulo se cargue
echo "blacklist nombre_modulo" | sudo tee /etc/modprobe.d/blacklist-nombre.conf
```

### Parámetros del Kernel en Tiempo Real (sysctl)

```bash
# Ver todos los parámetros
sysctl -a

# Ver un parámetro específico
sysctl net.ipv4.ip_forward

# Cambiar temporalmente
sudo sysctl -w net.ipv4.ip_forward=1

# Cambiar permanentemente
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-custom.conf
sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

### Actualizar el Kernel

```bash
# Debian/Ubuntu (se actualiza con apt)
sudo apt update && sudo apt upgrade

# Ver kernels disponibles
apt search linux-image

# Instalar un kernel específico
sudo apt install linux-image-6.1.0-xx-amd64

# Después de actualizar, reiniciar para usar el nuevo
sudo reboot

# Verificar que estás usando el nuevo
uname -r
```

> **En servidores de producción:** No actualizar el kernel sin probar en staging primero. Un kernel nuevo puede romper drivers o cambiar comportamiento.

---

## 5. Targets de Systemd (Equivalentes a Runlevels)

```bash
# Ver el target actual
systemctl get-default

# Cambiar target por defecto
sudo systemctl set-default multi-user.target    # Sin GUI (servidores)
sudo systemctl set-default graphical.target     # Con GUI

# Cambiar temporalmente (sin reiniciar)
sudo systemctl isolate multi-user.target        # Ir a modo texto
sudo systemctl isolate rescue.target            # Modo rescue
```

| Target              | Equivalente | Uso                             |
| ------------------- | ----------- | ------------------------------- |
| `poweroff.target`   | Runlevel 0  | Apagar                          |
| `rescue.target`     | Runlevel 1  | Modo rescue (single-user)       |
| `multi-user.target` | Runlevel 3  | Modo texto con red (servidores) |
| `graphical.target`  | Runlevel 5  | Modo gráfico                    |
| `reboot.target`     | Runlevel 6  | Reiniciar                       |

---

## 6. Troubleshooting de Boot

```bash
# Ver mensajes del kernel del boot actual
dmesg
dmesg | grep -i error

# Ver logs del boot
journalctl -b            # Boot actual
journalctl -b -1         # Boot anterior
journalctl --list-boots  # Lista de todos los boots

# Ver servicios que fallaron al arrancar
systemctl --failed

# Ver cuánto tarda cada servicio en iniciar
systemd-analyze
systemd-analyze blame              # Ordenado por tiempo
systemd-analyze critical-chain     # Cadena crítica
```
