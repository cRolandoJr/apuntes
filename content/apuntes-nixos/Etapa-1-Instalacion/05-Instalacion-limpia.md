# Instalación limpia paso a paso

> **Pre-requisito:** leer [[04-Btrfs-y-Particionado]] primero. Los pasos de particionado se asumen hechos.

## Antes de arrancar

- [ ] Bajar ISO de NixOS Plasma desde https://nixos.org/download (minimal o con Plasma)
- [ ] Flashear a USB con `dd` o Ventoy
- [ ] Bootear desde el USB en el Victus (F9 o F12 para boot menu)
- [ ] Conectarse a internet (WiFi desde el instalador gráfico, o cable)

---

## Opción A — Instalador gráfico (recomendado para el primer boot)

El instalador gráfico de NixOS Plasma hace el particionado y configuración básica automáticamente. Después podés reemplazar `configuration.nix` con tu config manual.

1. Seleccionar idioma, zona horaria, teclado
2. En particionado: elegir **Manual** y replicar el esquema de [[04-Btrfs-y-Particionado]]
3. Crear usuario con contraseña
4. Finalizar — el instalador corre `nixos-install` por vos

---

## Opción B — Instalación manual desde TTY (más control)

### 1. Particionar y montar (ver [[04-Btrfs-y-Particionado]])

```bash
# Verificar que todo esté montado correctamente
mount | grep /mnt
df -h /mnt /mnt/nix /mnt/boot/efi
```

### 2. Generar configuración inicial

```bash
# Genera configuration.nix y hardware-configuration.nix
nixos-generate-config --root /mnt

# Revisar lo generado
cat /mnt/etc/nixos/hardware-configuration.nix
cat /mnt/etc/nixos/configuration.nix
```

### 3. Editar configuration.nix

```bash
nano /mnt/etc/nixos/configuration.nix
```

Configuración mínima funcional para el primer boot:

```nix
{ config, pkgs, lib, ... }:
{
  imports = [ ./hardware-configuration.nix ];

  # Bootloader UEFI
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  # Kernel — el genérico funciona bien con AMD
  boot.kernelPackages = pkgs.linuxPackages_latest;

  # Red
  networking.hostName = "victus";
  networking.networkmanager.enable = true;

  # Zona horaria
  time.timeZone = "America/Argentina/Buenos_Aires";

  # Localización
  i18n.defaultLocale = "es_AR.UTF-8";
  i18n.extraLocaleSettings = {
    LC_ALL = "es_AR.UTF-8";
  };
  console.keyMap = "la-latin1";

  # Firmware AMD (importante para tu hardware)
  hardware.enableRedistributableFirmware = true;
  hardware.cpu.amd.updateMicrocode = true;

  # GPU AMD
  hardware.graphics.enable = true;

  # Sonido
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
  };

  # Entorno de escritorio temporal (Plasma para el primer boot)
  services.xserver.enable = true;
  services.displayManager.sddm.enable = true;
  services.desktopManager.plasma6.enable = true;

  # Usuario
  users.users.rolando = {
    isNormalUser = true;
    description = "Rolando";
    extraGroups = [ "networkmanager" "wheel" "video" "audio" ];
    shell = pkgs.fish;
    # Contraseña temporal — cambiar con passwd después del boot
    initialPassword = "cambiar";
  };

  # Paquetes mínimos
  environment.systemPackages = with pkgs; [
    wget
    curl
    git
    neovim
    fish
    htop
    pciutils    # lspci — útil para verificar hardware
    usbutils    # lsusb
  ];

  # Nix — habilitar Flakes desde el día 1
  nix.settings = {
    experimental-features = [ "nix-command" "flakes" ];
    auto-optimise-store = true;    # deduplicar archivos idénticos en la store
  };

  # Garbage collection automático
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 30d";
  };

  # Permitir software propietario (drivers, etc.)
  nixpkgs.config.allowUnfree = true;

  system.stateVersion = "25.05";    # poner la versión actual de NixOS
}
```

### 4. Instalar

```bash
nixos-install

# Te va a pedir contraseña de root al final
# El usuario tiene initialPassword = "cambiar", cambiarla con passwd después
```

### 5. Reinicar

```bash
reboot
# Sacar el USB cuando reinicie
```

---

## Post-instalación inmediata

```bash
# Verificar que el sistema arrancó correctamente
systemctl status

# Cambiar contraseña de usuario
passwd rolando

# Verificar red
ping -c 3 google.com

# Verificar que Nix funciona
nix --version
nixos-version

# Ver las generaciones disponibles
nixos-rebuild list-generations
# o en GRUB al bootear (entradas anteriores)
```

---

## Verificar hardware AMD

```bash
# GPU
lspci | grep -i amd
glxinfo | grep "OpenGL renderer"    # requiere mesa-demos

# CPU
lscpu | grep -E "Model name|Thread|Core"

# Temperatura (instalar lm_sensors)
sensors

# NVMe
nvme list
nvme smart-log /dev/nvme0n1    # health del SSD
```

---

## Primer nixos-rebuild después del boot

Una vez en el sistema, el flujo de trabajo normal:

```bash
# Editar la configuración
sudo nano /etc/nixos/configuration.nix

# Aplicar cambios
sudo nixos-rebuild switch

# Si algo falla y el sistema no bootea → en GRUB elegir generación anterior
# Si arrancó pero querés revertir:
sudo nixos-rebuild switch --rollback
```

---

## Checkpoint — qué tenés después de esto

- NixOS instalado en WD SN850x con btrfs y subvolúmenes
- Plasma como escritorio temporal
- Flakes habilitados
- Usuario con fish shell
- Listo para los siguientes pasos: Flakes + Home Manager + Hyprland
