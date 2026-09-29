# Migración de Hyprland a Home Manager

## Estrategia: migración progresiva

No migrés todo de una vez. El proceso recomendado:

1. Instalar NixOS con Plasma (primer boot estable)
2. Agregar Hyprland al sistema pero sin borrar Plasma
3. Arrancar en Hyprland y verificar que funciona
4. Migrar la config de Arch a Home Manager de a partes
5. Cuando esté estable, eliminar Plasma

---

## Paso 1 — Habilitar Hyprland en el sistema

```nix
# configuration.nix
{
  # Mantener Plasma como fallback durante la migración
  services.displayManager.sddm.enable = true;
  services.desktopManager.plasma6.enable = true;

  # Agregar Hyprland
  programs.hyprland = {
    enable = true;
    xwayland.enable = true;    # compatibilidad con apps X11
  };

  # Variables de entorno necesarias para Wayland
  environment.sessionVariables = {
    NIXOS_OZONE_WL = "1";       # Chrome/Electron en Wayland nativo
    WLR_NO_HARDWARE_CURSORS = "1";  # fix para algunos GPUs AMD
  };
}
```

---

## Paso 2 — Config de Hyprland en Home Manager

En Home Manager podés gestionar Hyprland de dos formas:

### Opción A: config como archivo (recomendada al migrar)

Copiás tu `hyprland.conf` actual de Arch a tu repo Nix y apuntás a él:

```nix
# home.nix
wayland.windowManager.hyprland = {
  enable = true;
  # Usar tu archivo de config existente directamente
  # Crea un symlink en ~/.config/hypr/hyprland.conf
};

# Copiar el archivo de config tal cual
home.file.".config/hypr/hyprland.conf".source = ./dotfiles/hypr/hyprland.conf;
```

Esto te permite copiar la config de Arch sin reescribirla en Nix. La migrás progresivamente.

### Opción B: config declarativa en Nix (destino final)

```nix
# home.nix
wayland.windowManager.hyprland = {
  enable = true;

  settings = {
    # Monitor
    monitor = [
      ",preferred,auto,1"    # auto-detectar
      # "eDP-1,1920x1080@144,0x0,1"  # si querés especificar
    ];

    # Variables de entorno de Hyprland
    env = [
      "XCURSOR_SIZE,24"
      "HYPRCURSOR_SIZE,24"
    ];

    # General
    general = {
      gaps_in = 5;
      gaps_out = 10;
      border_size = 2;
      "col.active_border" = "rgba(33ccffee) rgba(00ff99ee) 45deg";
      "col.inactive_border" = "rgba(595959aa)";
      layout = "dwindle";
    };

    # Decoraciones
    decoration = {
      rounding = 10;
      blur = {
        enabled = true;
        size = 3;
        passes = 1;
      };
      drop_shadow = true;
      shadow_range = 4;
      shadow_render_power = 3;
    };

    # Animaciones
    animations = {
      enabled = true;
      bezier = "myBezier, 0.05, 0.9, 0.1, 1.05";
      animation = [
        "windows, 1, 7, myBezier"
        "windowsOut, 1, 7, default, popin 80%"
        "border, 1, 10, default"
        "fade, 1, 7, default"
        "workspaces, 1, 6, default"
      ];
    };

    # Tecla modificadora
    "$mod" = "SUPER";

    # Keybindings
    bind = [
      "$mod, Q, killactive,"
      "$mod, M, exit,"
      "$mod, E, exec, dolphin"
      "$mod, V, togglefloating,"
      "$mod, R, exec, wofi --show drun"
      "$mod, P, pseudo,"
      "$mod, J, togglesplit,"
      # Mover foco
      "$mod, left, movefocus, l"
      "$mod, right, movefocus, r"
      "$mod, up, movefocus, u"
      "$mod, down, movefocus, d"
      # Workspaces
      "$mod, 1, workspace, 1"
      "$mod, 2, workspace, 2"
      "$mod, 3, workspace, 3"
      "$mod, 4, workspace, 4"
      "$mod, 5, workspace, 5"
      # Mover ventana a workspace
      "$mod SHIFT, 1, movetoworkspace, 1"
      "$mod SHIFT, 2, movetoworkspace, 2"
      "$mod SHIFT, 3, movetoworkspace, 3"
    ];

    # Apps al arrancar
    exec-once = [
      "waybar"
      "hyprpaper"
      "dunst"
    ];
  };
};
```

---

## Apps esenciales para Hyprland en NixOS

```nix
# home.nix — paquetes de un setup Hyprland completo
home.packages = with pkgs; [
  # Bar
  waybar

  # Launcher
  wofi         # o rofi-wayland
  fuzzel       # alternativa minimalista

  # Notificaciones
  dunst
  libnotify    # para enviar notificaciones desde scripts

  # Wallpaper
  hyprpaper
  swww         # wallpaper animado

  # Screenshot
  grim         # captura
  slurp        # seleccionar región
  swappy       # anotaciones

  # Clipboard
  wl-clipboard
  cliphist

  # Lock screen
  hyprlock

  # Idle
  hypridle

  # Portal (necesario para screensharing, file picker)
  xdg-desktop-portal-hyprland

  # Gestor de archivos
  nautilus     # o dolphin, thunar

  # Visor de imágenes
  imv

  # Reproductor de video
  mpv

  # Polkit (autenticación GUI)
  polkit_gnome
];
```

---

## Waybar en Home Manager

```nix
# home.nix
programs.waybar = {
  enable = true;
  style = builtins.readFile ./dotfiles/waybar/style.css;   # tu CSS existente
  settings = {
    mainBar = {
      layer = "top";
      position = "top";
      height = 30;
      modules-left = [ "hyprland/workspaces" "hyprland/mode" ];
      modules-center = [ "hyprland/window" ];
      modules-right = [ "pulseaudio" "network" "cpu" "memory" "clock" "tray" ];

      "hyprland/workspaces" = {
        disable-scroll = true;
        all-outputs = true;
      };

      clock = {
        format = "{:%H:%M}";
        format-alt = "{:%Y-%m-%d %H:%M}";
        tooltip-format = "<big>{:%Y %B}</big>\n<tt><small>{calendar}</small></tt>";
      };

      cpu = {
        format = " {usage}%";
        tooltip = false;
      };

      memory = {
        format = " {}%";
      };

      network = {
        format-wifi = " {signalStrength}%";
        format-ethernet = " {ifname}";
        format-disconnected = "⚠ Disconnected";
      };
    };
  };
};
```

---

## Migrar la config existente de Arch — proceso recomendado

```bash
# En Arch, antes de instalar NixOS
# Hacer backup de toda la config de Hyprland
cp -r ~/.config/hypr/ ~/backup-hypr/
cp -r ~/.config/waybar/ ~/backup-waybar/
cp -r ~/.config/dunst/ ~/backup-dunst/
cp -r ~/.config/wofi/ ~/backup-wofi/

# Subir a tu repo de NixOS como dotfiles/
# (para copiar con home.file.".config/hypr".source = ./dotfiles/hypr)
```

**Orden de migración:**

1. Primero: copiar archivos tal cual con `home.file` → funcionando en NixOS
2. Después: mover configuraciones a módulos declarativos de HM (`programs.waybar`, etc.) de a uno
3. Nunca migrar todo de una vez — si algo se rompe no sabés qué fue

---

## Pantalla negra / problemas al iniciar Hyprland

```bash
# Ver logs de Hyprland
cat /tmp/hypr/$(ls /tmp/hypr)/hyprland.log | tail -50

# Variables importantes para AMD en NixOS
# Agregar en configuration.nix:
environment.sessionVariables = {
  WLR_RENDERER = "vulkan";         # renderer Vulkan para mejor performance con AMD
  WLR_NO_HARDWARE_CURSORS = "1";   # fix cursor invisible en algunos setups
  LIBVA_DRIVER_NAME = "radeonsi";  # aceleración de video AMD
  MOZ_ENABLE_WAYLAND = "1";        # Firefox en Wayland nativo
};
```
