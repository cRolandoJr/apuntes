# Particionado con btrfs para NixOS

## Por qué btrfs para NixOS

NixOS genera muchos archivos pequeños con rutas largas en `/nix/store`. btrfs maneja esto mejor que ext4:

| Característica | Beneficio |
|----------------|-----------|
| **Copy-on-Write (CoW)** | Generaciones del sistema comparten bloques idénticos — menos espacio real |
| **Snapshots instantáneos** | Antes de `nixos-rebuild`, snapshotear. Revertís desde GRUB sin depender de Nix |
| **Compresión zstd** | ~30-40% menos uso de disco, CPU moderno no lo nota |
| **Subvolúmenes** | Separar `@` (root) de `@snapshots` sin particiones físicas distintas |

---

## Esquema de particiones recomendado

Para tu Victus con SSD NVMe (WD SN850x):

```
/dev/nvme0n1
├── nvme0n1p1   512MB    FAT32    /boot/efi    (partición EFI)
└── nvme0n1p2   resto    btrfs    (volumen btrfs con subvolúmenes)
```

**Sin swap en disco** — con 32 GB de RAM no la necesitás salvo para hibernación. Si querés hibernación, usá un swapfile dentro de btrfs (ver más abajo).

---

## Subvolúmenes btrfs recomendados

```
volumen btrfs
├── @           →  /              (sistema y home juntos, como pediste)
├── @nix        →  /nix           (store de Nix — excluida de snapshots del sistema)
├── @snapshots  →  /.snapshots    (snapshots de @)
└── @swap       →  /swap          (swapfile, solo si querés hibernación)
```

### Por qué separar `/nix` en su propio subvolumen

La Nix Store puede crecer mucho y no tiene sentido incluirla en los snapshots — si revertís un snapshot no querés perder paquetes descargados. Tenerla en `@nix` permite excluirla de los snapshots de `@` sin perder nada.

---

## Paso a paso — desde la ISO

### 1. Identificar el disco

```bash
lsblk                     # ver todos los discos
# El WD SN850x va a aparecer como /dev/nvme0n1 o similar
```

### 2. Crear las particiones

```bash
# Abrir el disco con gdisk (GPT, necesario para UEFI)
gdisk /dev/nvme0n1

# Dentro de gdisk:
# o → crear nueva tabla GPT (borra todo)
# n → nueva partición
#   número: 1
#   primer sector: enter (default)
#   último sector: +512M
#   tipo: ef00  (EFI System)
# n → nueva partición
#   número: 2
#   primer sector: enter
#   último sector: enter (todo el resto)
#   tipo: 8300  (Linux filesystem)
# w → escribir y salir
```

### 3. Formatear

```bash
# Partición EFI
mkfs.fat -F 32 -n BOOT /dev/nvme0n1p1

# Partición btrfs
mkfs.btrfs -L nixos /dev/nvme0n1p2
```

### 4. Crear los subvolúmenes

```bash
# Montar el volumen raíz temporalmente
mount /dev/nvme0n1p2 /mnt

# Crear subvolúmenes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@nix
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap    # solo si querés hibernación

# Desmontar
umount /mnt
```

### 5. Montar con las opciones correctas

```bash
# Opciones de montaje que vas a usar en todos los subvolúmenes btrfs:
# compress=zstd:3   → compresión zstd nivel 3 (balance velocidad/ratio)
# noatime           → no actualizar tiempo de acceso en cada lectura (SSD friendly)
# ssd               → optimizaciones para SSD
# space_cache=v2    → cache de espacio libre mejorado

# Montar subvolumen raíz
mount -o subvol=@,compress=zstd:3,noatime,ssd,space_cache=v2 /dev/nvme0n1p2 /mnt

# Crear puntos de montaje
mkdir -p /mnt/{boot/efi,nix,.snapshots,swap}

# Montar /nix
mount -o subvol=@nix,compress=zstd:3,noatime,ssd,space_cache=v2 /dev/nvme0n1p2 /mnt/nix

# Montar /.snapshots
mount -o subvol=@snapshots,compress=zstd:3,noatime,ssd,space_cache=v2 /dev/nvme0n1p2 /mnt/.snapshots

# Montar EFI
mount /dev/nvme0n1p1 /mnt/boot/efi
```

### 6. Swapfile en btrfs (opcional — solo para hibernación)

```bash
# Montar @swap
mount -o subvol=@swap,noatime /dev/nvme0n1p2 /mnt/swap

# El swapfile en btrfs NO puede estar comprimido ni con CoW
# Crear el archivo con las opciones correctas
btrfs filesystem mkswapfile --size 32g /mnt/swap/swapfile

# Activar
swapon /mnt/swap/swapfile
```

---

## Cómo queda en hardware-configuration.nix

Después de `nixos-generate-config`, vas a ver algo así:

```nix
fileSystems."/" = {
  device = "/dev/disk/by-uuid/xxxx-xxxx";
  fsType = "btrfs";
  options = [ "subvol=@" "compress=zstd:3" "noatime" "ssd" "space_cache=v2" ];
};

fileSystems."/nix" = {
  device = "/dev/disk/by-uuid/xxxx-xxxx";   # mismo UUID, mismo disco
  fsType = "btrfs";
  options = [ "subvol=@nix" "compress=zstd:3" "noatime" "ssd" "space_cache=v2" ];
};

fileSystems."/.snapshots" = {
  device = "/dev/disk/by-uuid/xxxx-xxxx";
  fsType = "btrfs";
  options = [ "subvol=@snapshots" "compress=zstd:3" "noatime" "ssd" "space_cache=v2" ];
};

fileSystems."/boot/efi" = {
  device = "/dev/disk/by-uuid/yyyy-yyyy";
  fsType = "vfat";
  options = [ "fmask=0077" "dmask=0077" ];
};
```

Usá siempre `by-uuid` — si cambiás el disco a otro puerto SATA/NVMe, el UUID no cambia pero `/dev/nvme0n1` sí.

---

## Snapshots manuales con snapper (post-instalación)

```bash
# Instalar snapper para gestionar snapshots automáticamente
# En configuration.nix:
services.snapper = {
  snapshotInterval = "hourly";
  configs.root = {
    subvolume = "/";
    extraConfig = ''
      ALLOW_USERS="rolando"
      TIMELINE_CREATE=yes
      TIMELINE_CLEANUP=yes
      TIMELINE_LIMIT_HOURLY=5
      TIMELINE_LIMIT_DAILY=7
      TIMELINE_LIMIT_WEEKLY=4
      TIMELINE_LIMIT_MONTHLY=3
    '';
  };
};

# Comandos manuales
snapper -c root create --description "antes de cambio importante"
snapper -c root list
snapper -c root delete 3
```
