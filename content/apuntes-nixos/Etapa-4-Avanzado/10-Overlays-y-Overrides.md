# Overlays y Overrides — personalizar paquetes

## Cuándo necesitás esto

- Un paquete en nixpkgs está en una versión vieja y querés una más nueva
- Necesitás compilar un paquete con flags distintos
- Un paquete tiene un bug y querés aplicar un parche
- Querés agregar un paquete que no existe en nixpkgs

---

## Override — modificar un paquete existente

```nix
# Cambiar las opciones de compilación de un paquete
pkgs.neovim.override {
  withPython3 = true;
  withRuby = false;
}

# En configuration.nix
environment.systemPackages = [
  (pkgs.neovim.override { withPython3 = true; })
];
```

`overrideAttrs` — modificar los atributos de la derivación (más potente):

```nix
# Aplicar un parche o cambiar la versión
(pkgs.algun-paquete.overrideAttrs (old: {
  version = "1.2.3";
  src = pkgs.fetchFromGitHub {
    owner = "usuario";
    repo = "repo";
    rev = "v1.2.3";
    sha256 = "sha256-xxxx";   # nix hash file $(nix-prefetch-url ...)
  };
  # Agregar un parche
  patches = (old.patches or []) ++ [ ./mi-parche.patch ];
}))
```

---

## Overlays — modificaciones globales y persistentes

Un overlay es una función que recibe nixpkgs y devuelve un set con paquetes nuevos o modificados. Se aplica globalmente a todos los módulos.

```nix
# flake.nix o configuration.nix
nixpkgs.overlays = [
  # Overlay como función inline
  (final: prev: {
    # Sobreescribir un paquete con una versión modificada
    neovim = prev.neovim.override { withPython3 = true; };

    # Agregar un paquete que no existe en nixpkgs
    mi-herramienta = prev.callPackage ./pkgs/mi-herramienta.nix { };
  })
];
```

`final` es nixpkgs con el overlay aplicado (para referencias entre overlays).
`prev` es nixpkgs sin el overlay (para acceder al paquete original).

### Overlay en archivo separado

```nix
# overlays/default.nix
final: prev: {
  neovim = prev.neovim.override { withPython3 = true; };
}

# En flake.nix
nixpkgs.overlays = [ (import ./overlays) ];
```

---

## Empaquetar algo que no existe en nixpkgs

```nix
# pkgs/mi-app/default.nix
{ lib, buildGoModule, fetchFromGitHub }:

buildGoModule rec {
  pname = "mi-app";
  version = "1.0.0";

  src = fetchFromGitHub {
    owner = "usuario";
    repo = "mi-app";
    rev = "v${version}";
    sha256 = "sha256-xxxx";   # obtener con: nix-prefetch-url --unpack URL
  };

  vendorHash = "sha256-yyyy";  # hash del vendor/ generado

  meta = {
    description = "Mi aplicación";
    license = lib.licenses.mit;
    maintainers = [ ];
  };
}

# Usar en configuration.nix o home.nix
environment.systemPackages = [
  (pkgs.callPackage ./pkgs/mi-app { })
];
```

---

## nixpkgs con versión específica (pin manual)

Si necesitás un paquete de una versión específica de nixpkgs sin actualizar todo:

```nix
# flake.nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  # Pinear otra versión de nixpkgs para paquetes específicos
  nixpkgs-stable.url = "github:NixOS/nixpkgs/nixos-25.05";
};

outputs = { self, nixpkgs, nixpkgs-stable, ... }:
let
  pkgs-stable = nixpkgs-stable.legacyPackages.x86_64-linux;
in {
  nixosConfigurations.victus = nixpkgs.lib.nixosSystem {
    modules = [
      ./configuration.nix
      {
        environment.systemPackages = [
          pkgs-stable.algun-paquete-estable   # este viene de la versión estable
        ];
      }
    ];
  };
};
```
