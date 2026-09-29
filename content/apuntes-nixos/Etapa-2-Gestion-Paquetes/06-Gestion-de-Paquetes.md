# Gestión de Paquetes — nix-env vs nix profile vs declarativo

## El problema de las tres formas

NixOS tiene varias formas de instalar paquetes y es confuso al principio. La respuesta corta:

> **Usá siempre el modo declarativo en `configuration.nix` o Home Manager. Las otras dos formas existen por razones históricas y tienen desventajas importantes.**

---

## Comparación

| Método | Comando | Estado | Para usar cuando |
|--------|---------|--------|-----------------|
| `nix-env` | `nix-env -iA nixpkgs.firefox` | Imperativo, por usuario | Nunca (legacy) |
| `nix profile` | `nix profile install nixpkgs#firefox` | Imperativo, por usuario | Probar algo rápido y temporal |
| `configuration.nix` | declarar en `environment.systemPackages` | Declarativo, sistema | Paquetes del sistema para todos los usuarios |
| `home-manager` | declarar en `home.packages` | Declarativo, por usuario | Paquetes específicos de tu usuario |

---

## `nix-env` — el modo legacy (evitar)

```bash
# Instalar
nix-env -iA nixpkgs.firefox

# Desinstalar
nix-env -e firefox

# Listar instalados
nix-env -q

# Actualizar todo
nix-env -u
```

**Por qué evitarlo:**
- El estado no está en ningún archivo — si perdés el disco, no sabés qué tenías instalado
- Genera un perfil *por usuario* que no está sincronizado con el sistema declarativo
- Puede crear conflictos con paquetes del sistema
- Es imperativo — contradice la filosofía de NixOS

---

## `nix profile` — el modo imperativo moderno

```bash
# Instalar (usa la sintaxis de Flakes)
nix profile install nixpkgs#firefox

# Listar
nix profile list

# Actualizar
nix profile upgrade firefox

# Desinstalar
nix profile remove firefox
```

**Cuándo usarlo:** para probar un paquete sin toccar `configuration.nix`. Por ejemplo, querés ver si `neovide` funciona bien antes de agregarlo permanentemente.

**Problema:** igual que `nix-env`, el estado no está documentado en ningún archivo de configuración.

---

## Modo declarativo — el correcto

### Paquetes del sistema (todos los usuarios)

```nix
# /etc/nixos/configuration.nix
environment.systemPackages = with pkgs; [
  wget
  curl
  git
  neovim
  htop
  fish
];
```

```bash
sudo nixos-rebuild switch
```

### Paquetes de usuario (Home Manager)

```nix
# ~/.config/home-manager/home.nix
home.packages = with pkgs; [
  firefox
  discord
  obsidian
  spotify
];
```

```bash
home-manager switch
```

---

## `nix run` — ejecutar sin instalar

Ejecuta un paquete de nixpkgs directamente, sin instalarlo. Perfecto para herramientas que usás una vez:

```bash
# Ejecutar cowsay sin instalarlo
nix run nixpkgs#cowsay -- "hola nix"

# Abrir una shell temporal con Python disponible
nix shell nixpkgs#python3
# → ahora python3 está en PATH dentro de esta shell
# → al salir (exit), desaparece

# Con múltiples paquetes
nix shell nixpkgs#python3 nixpkgs#poetry
```

**Esto es muy útil en el trabajo.** Si necesitás una herramienta para un task puntual no la instalás globalmente — la corrés con `nix run` o la agregás al `devShell` del proyecto.

---

## `nix-shell` / `nix develop` — entornos de desarrollo

Para proyectos con dependencias específicas que no querés mezclar con el sistema:

```bash
# Abrir una shell con las dependencias de un proyecto (Flakes)
nix develop

# Shell temporal con dependencias específicas (sin flake.nix)
nix shell nixpkgs#go nixpkgs#gopls nixpkgs#delve

# Legacy — usar un shell.nix
nix-shell
```

Esto es el equivalente a `venv` en Python o `node_modules` en Node, pero para cualquier herramienta y cualquier lenguaje. El entorno existe mientras tenés la shell abierta; al cerrarla, el sistema no cambia.

---

## Garbage Collection — limpiar la store

La Nix Store acumula paquetes de generaciones anteriores. El GC los limpia:

```bash
# Ver cuánto ocupa la store
du -sh /nix/store

# Eliminar generaciones antiguas y sus paquetes
nix-collect-garbage -d           # borra TODAS las generaciones viejas excepto la actual
nix-collect-garbage --delete-older-than 30d   # borra generaciones de más de 30 días

# Optimizar la store (hardlinks para archivos idénticos)
nix store optimise

# Configurar GC automático en configuration.nix
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 30d";
};
```

> **Advertencia:** `nix-collect-garbage -d` borra *todas* las generaciones anteriores. Si la nueva tiene un problema y borraste las viejas, no podés hacer rollback desde GRUB. Hacelo solo cuando estés seguro de que el sistema funciona bien.
