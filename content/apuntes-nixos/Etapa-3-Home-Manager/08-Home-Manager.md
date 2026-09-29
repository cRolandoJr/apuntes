# Home Manager — Configuración declarativa de usuario

## Por qué Home Manager

`configuration.nix` gestiona el sistema. Pero muchas cosas son específicas de tu usuario:
- Paquetes que solo vos usás (Discord, Spotify, herramientas de desarrollo)
- Dotfiles (`.zshrc`, `hyprland.conf`, `neovim/init.lua`, etc.)
- Variables de entorno de usuario
- Servicios de usuario (syncthing, mpd, etc.)

Sin Home Manager, esas cosas quedan fuera del sistema declarativo — las configurás a mano y se pierden si reinstalás.

**Home Manager lleva todo eso al modelo declarativo.** Tu `home.nix` describe exactamente cómo debe ser tu entorno de usuario. En cualquier máquina nueva, con ese archivo, tu entorno queda idéntico.

---

## Integración con el sistema (NixOS module)

Hay dos formas de usar Home Manager. La recomendada para NixOS es integrarla como módulo del sistema en `flake.nix`:

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.victus = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager   # ← integrar HM como módulo
        {
          home-manager.useGlobalPkgs = true;      # usa el mismo nixpkgs del sistema
          home-manager.useUserPackages = true;    # instala paquetes en el perfil del usuario
          home-manager.users.rolando = import ./home.nix;
        }
      ];
    };
  };
}
```

Con esta integración, `sudo nixos-rebuild switch` aplica tanto el sistema como Home Manager en un solo comando.

---

## Estructura de `home.nix`

```nix
{ config, pkgs, lib, ... }:

{
  # ─── IDENTIDAD ────────────────────────────────────────────────────────────
  home.username = "rolando";
  home.homeDirectory = "/home/rolando";

  # Versión de Home Manager — igual que system.stateVersion, no cambiar
  home.stateVersion = "25.05";

  # ─── PAQUETES DE USUARIO ──────────────────────────────────────────────────
  home.packages = with pkgs; [
    # Aplicaciones
    firefox
    discord
    obsidian
    spotify

    # Desarrollo
    go
    gopls           # LSP de Go
    delve           # debugger de Go
    python3
    uv              # gestor de proyectos Python

    # CLI
    ripgrep
    fd
    bat             # cat mejorado
    eza             # ls mejorado
    fzf
    zoxide          # cd inteligente
    lazygit
  ];

  # ─── DOTFILES GESTIONADOS POR HM ──────────────────────────────────────────
  # Opción 1: contenido inline
  home.file.".config/starship.toml".text = ''
    [character]
    success_symbol = "[❯](bold green)"
    error_symbol = "[❯](bold red)"
  '';

  # Opción 2: apuntar a un archivo en tu repo
  home.file.".config/neovim/init.lua".source = ./dotfiles/neovim/init.lua;

  # ─── VARIABLES DE ENTORNO ─────────────────────────────────────────────────
  home.sessionVariables = {
    EDITOR = "nvim";
    BROWSER = "firefox";
    GOPATH = "${config.home.homeDirectory}/go";
  };

  # ─── PROGRAMAS GESTIONADOS POR HM (con opciones declarativas) ─────────────
  # HM tiene módulos específicos para programas comunes que generan
  # la config automáticamente. Mucho mejor que gestionar dotfiles a mano.

  programs.git = {
    enable = true;
    userName = "Rolando";
    userEmail = "tu@email.com";
    extraConfig = {
      init.defaultBranch = "main";
      pull.rebase = true;
      core.editor = "nvim";
    };
    delta.enable = true;    # diff mejorado
  };

  programs.fish = {
    enable = true;
    shellAliases = {
      ls = "eza --icons";
      ll = "eza -la --icons";
      cat = "bat";
      cd = "z";             # zoxide
      lg = "lazygit";
    };
    interactiveShellInit = ''
      zoxide init fish | source
      starship init fish | source
    '';
  };

  programs.starship = {
    enable = true;
    # La config de starship va en home.file o en settings (ver docs)
  };

  programs.neovim = {
    enable = true;
    defaultEditor = true;
    # Para config compleja de neovim conviene usar home.file apuntando a tus archivos
  };

  # ─── SERVICIOS DE USUARIO ─────────────────────────────────────────────────
  services.syncthing.enable = true;

  # ─── LET HOME MANAGER GESTIONAR SÍ MISMO ─────────────────────────────────
  programs.home-manager.enable = true;
}
```

---

## Módulos de programas disponibles

Home Manager tiene módulos para decenas de programas. Generan la config automáticamente:

```bash
# Buscar en la documentación
# https://home-manager-options.extenda.io/  ← buscador de opciones HM
# https://nix-community.github.io/home-manager/options.xhtml
```

| Programa | Módulo HM | Qué configura automáticamente |
|----------|-----------|-------------------------------|
| git | `programs.git` | `.gitconfig` con todas las opciones |
| ssh | `programs.ssh` | `~/.ssh/config` |
| fish | `programs.fish` | `config.fish`, aliases, plugins |
| zsh | `programs.zsh` | `.zshrc`, plugins, completions |
| neovim | `programs.neovim` | init.vim/lua, plugins con Nix |
| tmux | `programs.tmux` | `.tmux.conf` |
| starship | `programs.starship` | `starship.toml` |
| alacritty | `programs.alacritty` | `alacritty.toml` |
| kitty | `programs.kitty` | `kitty.conf` |
| vscode | `programs.vscode` | Extensiones, settings.json |
| firefox | `programs.firefox` | Extensiones, user.js |

---

## Aplicar cambios de Home Manager

Con la integración en `flake.nix`:

```bash
# Sistema + Home Manager en un solo comando
sudo nixos-rebuild switch --flake .#victus

# Solo Home Manager (si no cambiaste configuration.nix)
home-manager switch --flake .#rolando
```

---

## Rollback de Home Manager

```bash
# Ver generaciones de Home Manager
home-manager generations

# Activar una generación anterior
home-manager switch --flake .#rolando --switch-generation 5
```
