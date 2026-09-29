# Debugging en NixOS

## El error más común: leer el mensaje completo

Los errores de Nix son verbosos. La información útil está casi siempre al **final**, no al principio.

```bash
# Patrón: el error real está después de "error:"
sudo nixos-rebuild switch --flake .#victus 2>&1 | grep -A 5 "error:"
```

---

## Errores de evaluación (syntax/type errors)

Estos ocurren antes de descargar o compilar nada — Nix no puede evaluar el archivo `.nix`.

```
error: undefined variable 'pks'
       at /etc/nixos/configuration.nix:42:5
```

→ Typo en el nombre. En este caso `pks` en lugar de `pkgs`.

```
error: attribute 'enble' missing
```

→ Typo en una opción de NixOS. Verificar en https://search.nixos.org/options

```
error: infinite recursion encountered
```

→ Una opción referencia a otra que la referencia de vuelta. Común cuando se usa `config.` dentro del mismo módulo que la define. Solución: usar `lib.mkDefault` o mover la lógica.

---

## `nix repl` — explorar interactivamente

La herramienta más útil para entender qué tiene nixpkgs y probar expresiones:

```bash
nix repl

# Cargar nixpkgs
nix-repl> :l <nixpkgs>

# Buscar un paquete
nix-repl> pkgs.firefox
# → «derivation /nix/store/xxx-firefox-124.drv»

# Ver atributos de un paquete
nix-repl> pkgs.firefox.version
# → "124.0"

nix-repl> pkgs.firefox.meta.description
# → "A web browser"

# Ver qué opciones tiene un módulo
nix-repl> :l <nixpkgs/nixos>
nix-repl> options.services.openssh
# → muestra todas las sub-opciones

# Probar una expresión
nix-repl> builtins.map (x: x * 2) [ 1 2 3 ]
# → [ 2 4 6 ]

# Salir
nix-repl> :q
```

---

## `nix build` — buildear sin activar

Antes de `nixos-rebuild switch`, verificar que la configuración buildea:

```bash
# Buildear sin activar
nix build .#nixosConfigurations.victus.config.system.build.toplevel

# Si falla, el error es más claro que con nixos-rebuild
# El resultado queda en ./result (symlink)
ls -la result/
```

---

## `nix log` — ver el log de compilación

Cuando un paquete falla al compilarse:

```bash
# Ver el log del último build fallido
nix log /nix/store/xxx.drv

# O directamente después de un fallo:
# Nix muestra la ruta del .drv en el error
# nix log /nix/store/yyy-paquete-fallido.drv
```

---

## `nix why-depends` — por qué está en el cierre

El "cierre" de un paquete es el paquete más todas sus dependencias transitivas. A veces la closure es enorme y querés saber por qué:

```bash
# Por qué firefox depende de X
nix why-depends nixpkgs#firefox nixpkgs#libpng

# Por qué el sistema tiene incluido X
nix why-depends /run/current-system nixpkgs#python3
```

---

## `nix store` — inspeccionar la store

```bash
# Listar las dependencias de un paquete (su closure)
nix-store -q --references /nix/store/xxx-firefox-124/

# Ver cuánto ocupa un paquete y su closure
nix path-info -rS nixpkgs#firefox | sort -k2 -n | tail -10
#                   ↑ recursivo  ↑ sort por tamaño

# Verificar integridad de la store
nix store verify --all
```

---

## `nixos-option` — ver opciones del sistema

```bash
# Ver el valor actual de una opción
nixos-option services.openssh.enable
# output:
# Value:
#   true
# Default:
#   false
# Description:
#   Whether to enable the OpenSSH secure shell daemon.

# Ver todas las sub-opciones
nixos-option services.openssh
```

---

## Problemas comunes y soluciones

### "Hash mismatch" al descargar

```
hash mismatch in fixed-output derivation:
  specified: sha256-xxxx
  got:       sha256-yyyy
```

El hash en la derivación no coincide con lo descargado. Actualizaste el `src` pero no el hash. Solución:

```bash
# Obtener el hash correcto
nix-prefetch-url --unpack URL_DEL_ARCHIVO
# o
nix hash file ./archivo-descargado
```

### Un servicio no arranca después de nixos-rebuild

```bash
# Ver el estado del servicio
systemctl status nombre-del-servicio

# Ver logs
journalctl -xeu nombre-del-servicio

# Verificar que la configuración del servicio es correcta
nixos-option services.nombre-del-servicio
```

### "collision between ... in ...": dos paquetes instalan el mismo archivo

```
error: collision between `/nix/store/aaa-paquete-a/bin/herramienta`
                    and  `/nix/store/bbb-paquete-b/bin/herramienta`
```

Dos paquetes quieren instalar un binario con el mismo nombre. Solución: instalar solo uno de los dos, o usar `lib.hiPrio` para que uno tenga precedencia:

```nix
environment.systemPackages = [
  (lib.hiPrio pkgs.paquete-preferido)
  pkgs.paquete-secundario
];
```

### El sistema usa mucho espacio

```bash
# Ver el tamaño de la store
du -sh /nix/store

# Ver las generaciones
nix profile history --profile /nix/var/nix/profiles/system

# Limpiar generaciones viejas
sudo nix-collect-garbage --delete-older-than 30d
sudo nix store optimise
```

### "experimental feature nix-command is disabled"

```bash
# Significa que Flakes no está habilitado
# En configuration.nix:
nix.settings.experimental-features = [ "nix-command" "flakes" ];
# Luego: sudo nixos-rebuild switch
```

---

## Flujo de debugging recomendado

```
1. Leer el error desde el final hacia arriba
      ↓
2. ¿Es un error de evaluación? → revisar syntax en el .nix
      ↓
3. ¿Es un error de build? → nix log /nix/store/xxx.drv
      ↓
4. ¿No entendés qué hace algo? → nix repl + :l <nixpkgs>
      ↓
5. ¿Un servicio falla? → journalctl -xeu nombre-servicio
      ↓
6. ¿Una opción no existe? → nixos-option / search.nixos.org
      ↓
7. Si nada funciona → buscar en discourse.nixos.org o el foro de NixOS
```
