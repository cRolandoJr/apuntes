# Anatomía de configuration.nix

## El sistema de módulos — cómo funciona

Antes de ver el archivo en sí, hay que entender cómo NixOS lo usa.

NixOS no tiene un único archivo de configuración gigante. Tiene un **sistema de módulos**: cada módulo es una función que recibe el contexto del sistema y devuelve un attribute set con opciones. NixOS fusiona todos esos sets en uno solo y construye el sistema.

`configuration.nix` es el módulo raíz — el punto de entrada. Desde ahí podés importar otros módulos.

```
configuration.nix
├── imports: [hardware-configuration.nix, ./modulos/usuarios.nix, ...]
├── opciones del sistema (networking, services, etc.)
└── NixOS fusiona todo → sistema completo
```

---

## Estructura del archivo

Un `configuration.nix` típico generado por el instalador:

```nix
# El archivo ES una función que recibe el contexto del sistema
{ config, pkgs, lib, ... }:

{
  # ─── IMPORTS ──────────────────────────────────────────────────────────────
  imports = [
    # Generado automáticamente — detecta tu hardware (discos, red, etc.)
    # NO editar a mano salvo que sepas lo que hacés
    ./hardware-configuration.nix
  ];

  # ─── BOOTLOADER ───────────────────────────────────────────────────────────
  boot.loader.systemd-boot.enable = true;       # para sistemas UEFI
  boot.loader.efi.canTouchEfiVariables = true;  # permite que NixOS escriba en la partición EFI

  # ─── NETWORKING ───────────────────────────────────────────────────────────
  networking.hostName = "nixos";                # nombre del equipo en la red
  networking.networkmanager.enable = true;      # gestor de red con GUI

  # ─── LOCALIZACIÓN ─────────────────────────────────────────────────────────
  time.timeZone = "America/Argentina/Buenos_Aires";
  i18n.defaultLocale = "es_AR.UTF-8";
  console.keyMap = "la-latin1";                 # teclado latinoamericano en TTY

  # ─── ENTORNO DE ESCRITORIO ────────────────────────────────────────────────
  services.xserver.enable = true;               # servidor de display (X11)
  services.displayManager.sddm.enable = true;   # gestor de login
  services.desktopManager.plasma6.enable = true;

  # ─── SOUND ────────────────────────────────────────────────────────────────
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    pulse.enable = true;    # compatibilidad PulseAudio
  };

  # ─── USUARIOS ─────────────────────────────────────────────────────────────
  users.users.rolando = {
    isNormalUser = true;
    description = "Rolando";
    extraGroups = [ "networkmanager" "wheel" ];  # wheel = sudo
    shell = pkgs.fish;
  };

  # ─── PAQUETES DEL SISTEMA ─────────────────────────────────────────────────
  # Paquetes disponibles para TODOS los usuarios
  # Para paquetes de un usuario específico → Home Manager
  environment.systemPackages = with pkgs; [
    wget
    curl
    git
    neovim
    htop
  ];

  # ─── SERVICIOS ────────────────────────────────────────────────────────────
  services.openssh.enable = true;

  # ─── NIXPKGS ──────────────────────────────────────────────────────────────
  nixpkgs.config.allowUnfree = true;   # permite paquetes con licencia no libre (drivers, etc.)

  # ─── VERSIÓN DEL SISTEMA ──────────────────────────────────────────────────
  # MUY IMPORTANTE: determina desde qué versión aplican los defaults de NixOS
  # NO cambiar después de la instalación sin leer el changelog
  # NO tiene que coincidir exactamente con la versión instalada — es una promesa de compatibilidad
  system.stateVersion = "25.05";
}
```

---

## Los argumentos de la función: `config`, `pkgs`, `lib`

| Argumento | Qué es | Cuándo se usa |
|-----------|--------|---------------|
| `pkgs` | El repositorio de paquetes de nixpkgs | Siempre — para instalar cualquier cosa |
| `config` | El attribute set completo del sistema una vez evaluado | Para leer opciones de otros módulos y hacer lógica condicional |
| `lib` | Librería de funciones de nixpkgs | Para manipulación de strings, listas, sets, condicionales |
| `...` | Permite que NixOS pase argumentos extra sin error | Siempre debe estar |

Ejemplo de uso de `config` para lógica condicional:

```nix
{ config, pkgs, lib, ... }:
{
  # Solo instalar herramientas de virtualización si KVM está habilitado
  environment.systemPackages = with pkgs; [
    git
  ] ++ lib.optionals config.virtualisation.libvirtd.enable [
    virt-manager
    qemu
  ];
}
```

---

## `hardware-configuration.nix` — no tocar

Este archivo lo genera el instalador (`nixos-generate-config`) leyendo tu hardware real. Contiene:

```nix
{
  # Módulos del kernel para tu hardware específico
  boot.initrd.availableKernelModules = [ "nvme" "xhci_pci" "ahci" "usb_storage" ];

  # Tus particiones — generado desde /etc/fstab durante la instalación
  fileSystems."/" = {
    device = "/dev/disk/by-uuid/xxxx-xxxx";
    fsType = "btrfs";
    options = [ "subvol=@" "compress=zstd" "noatime" ];
  };

  fileSystems."/boot" = {
    device = "/dev/disk/by-uuid/yyyy-yyyy";
    fsType = "vfat";
  };

  swapDevices = [ ];

  # Detectado automáticamente
  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
  hardware.cpu.amd.updateMicrocode = lib.mkDefault config.hardware.enableRedistributableFirmware;
}
```

---

## El sistema de opciones — cómo saber qué existe

NixOS tiene miles de opciones. La forma de encontrarlas:

```bash
# Buscar opciones desde la línea de comandos
nixos-option services.openssh        # muestra todas las sub-opciones de openssh
nixos-option services.openssh.enable # muestra tipo, default, descripción

# O en el browser:
# https://search.nixos.org/options
```

Cada opción tiene:
- **Tipo:** `bool`, `string`, `list of string`, `attribute set`, etc.
- **Default:** valor que tiene si no lo especificás
- **Description:** qué hace

---

## `system.stateVersion` — el campo más confuso

```nix
system.stateVersion = "25.05";
```

**No es la versión de NixOS que tenés instalada.** Es la versión en la que *instalaste por primera vez* el sistema. Algunos servicios cambian su configuración por defecto entre versiones, y este campo le dice a NixOS desde qué versión aplicar los defaults.

**Regla:** ponelo una vez durante la instalación y no lo toques más, aunque actualices nixpkgs.

---

## Modularizar la configuración

Cuando `configuration.nix` crece, se divide en archivos:

```nix
# configuration.nix
{ config, pkgs, lib, ... }:
{
  imports = [
    ./hardware-configuration.nix
    ./modulos/usuarios.nix
    ./modulos/desarrollo.nix
    ./modulos/escritorio.nix
  ];

  # Solo lo que es verdaderamente global
  networking.hostName = "victus";
  system.stateVersion = "25.05";
}
```

```nix
# modulos/desarrollo.nix
{ pkgs, ... }:
{
  environment.systemPackages = with pkgs; [
    go
    python3
    docker
    git
  ];

  virtualisation.docker.enable = true;
}
```

Cada módulo es independiente. NixOS los fusiona todos automáticamente.
