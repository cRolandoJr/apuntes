# Canales, Flakes y nixpkgs

## El problema que Flakes resuelve

Antes de Flakes, NixOS usaba **canales** para gestionar las versiones de nixpkgs. Los canales tienen un problema fundamental: son **globales e implícitos**.

```bash
# Sin Flakes — el sistema depende del canal que tengas en ese momento
nix-channel --add https://nixos.org/channels/nixos-unstable nixos
nix-channel --update    # actualiza nixpkgs globalmente
```

Esto significa que dos máquinas con el "mismo" `configuration.nix` pueden tener versiones distintas de paquetes dependiendo de cuándo se ejecutó `nix-channel --update`. No es reproducible.

---

## Canales — el sistema legacy

```bash
# Ver canales actuales
nix-channel --list

# Canales principales
# nixos-25.05           → release estable (LTS, actualización de seguridad)
# nixos-unstable        → rolling release (paquetes más nuevos, puede tener bugs)
# nixpkgs-unstable      → como unstable pero sin la infra de NixOS

# Cambiar a unstable
nix-channel --add https://nixos.org/channels/nixos-unstable nixos
nix-channel --update
nixos-rebuild switch
```

**Cuándo usar canales:** cuando empezás y no querés la complejidad extra de Flakes. Pero cambiá a Flakes pronto.

---

## Flakes — el sistema moderno

Un Flake es un proyecto Nix con una **estructura estándar y reproducible**. Tiene dos archivos clave:

```
mi-nixos/
├── flake.nix     ← define entradas (inputs) y salidas (outputs)
└── flake.lock    ← pinea las versiones exactas de todas las entradas
```

El `flake.lock` es como el `go.sum` o el `package-lock.json` — garantiza que en cualquier máquina se use exactamente la misma versión de nixpkgs.

### `flake.nix` mínimo para NixOS

```nix
{
  description = "Configuración NixOS de Rolando";

  inputs = {
    # La "fuente de verdad" de todos los paquetes
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    # Home Manager — para la configuración de usuario
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";   # usa el mismo nixpkgs que el sistema
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.victus = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.rolando = import ./home.nix;
        }
      ];
    };
  };
}
```

### `flake.lock` — no editar a mano

```bash
# Actualizar todas las entradas (equivalente a pacman -Syu)
nix flake update

# Actualizar solo nixpkgs
nix flake update nixpkgs

# Ver qué va a cambiar antes de actualizar
nix flake update --dry-run

# Rebuild con el flake actualizado
sudo nixos-rebuild switch --flake .#victus
```

---

## Estructura de un repo de configuración NixOS con Flakes

Estructura recomendada cuando tenés Flakes + Home Manager:

```
~/.config/nixos/           (o ~/nixos/, donde prefieras)
├── flake.nix              ← entradas y salidas
├── flake.lock             ← versiones pineadas (commitear)
├── configuration.nix      ← sistema (bootloader, servicios, usuarios)
├── hardware-configuration.nix  ← generado por nixos-generate-config
├── home.nix               ← Home Manager (paquetes de usuario, dotfiles)
└── modules/               ← módulos opcionales
    ├── desarrollo.nix
    ├── escritorio.nix
    └── hyprland.nix
```

Esto va en un **repositorio Git**. La configuración completa del sistema queda versionada y reproducible.

---

## nixpkgs — el repositorio de paquetes

nixpkgs es el repositorio de paquetes de NixOS. Con más de 100.000 paquetes, es uno de los repositorios de software más grandes del mundo.

```bash
# Buscar un paquete
nix search nixpkgs firefox
nix search nixpkgs#python3   # formato Flakes

# O en el browser: https://search.nixos.org/packages

# Ver el código de un paquete (cómo está construido)
nix edit nixpkgs#firefox
```

### Canales de nixpkgs

| Canal | Estabilidad | Cuándo usar |
|-------|-------------|-------------|
| `nixos-25.05` | Estable | Servidores, producción |
| `nixos-unstable` | Rolling | Desktop, desarrollo — paquetes más nuevos |
| `nixpkgs-unstable` | Rolling | Como unstable pero solo paquetes, sin módulos NixOS |

Para tu setup de desarrollo en laptop: **nixos-unstable** es la recomendación. Tiene versiones más recientes de herramientas (Go, Python, etc.) y los problemas se corrigen rápido.

---

## Comandos Flakes que vas a usar seguido

```bash
# Aplicar cambios al sistema
sudo nixos-rebuild switch --flake .#victus

# Con mensaje de commit automático para el registro de generaciones
sudo nixos-rebuild switch --flake .#victus -v

# Solo buildear sin activar (para verificar que no hay errores)
sudo nixos-rebuild build --flake .#victus

# Actualizar flake.lock
nix flake update

# Ver info del flake
nix flake show
nix flake metadata

# Ingresar a un entorno de desarrollo de un proyecto remoto
nix develop github:usuario/repo

# Correr una app directamente desde GitHub
nix run github:usuario/repo
```

---

## Habilitar Flakes en configuration.nix

Si no usás el flake desde el inicio (instalación gráfica), habilitalo:

```nix
# configuration.nix
nix.settings = {
  experimental-features = [ "nix-command" "flakes" ];
};
```

Después de `nixos-rebuild switch`, podés usar `nix flake` y todos los comandos modernos.
