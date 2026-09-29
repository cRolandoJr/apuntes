# Rollbacks y Generaciones

## Cómo funciona el sistema de generaciones

Cada vez que corrés `nixos-rebuild switch`, NixOS crea una nueva **generación** — una foto completa del sistema. Las generaciones anteriores se mantienen en `/nix/var/nix/profiles/system-*`.

```bash
# Ver todas las generaciones
nixos-rebuild list-generations
# o
nix profile history --profile /nix/var/nix/profiles/system

# Output ejemplo:
# Generation 47  2026-05-14 10:23:01  (current)
# Generation 46  2026-05-13 20:11:42
# Generation 45  2026-05-12 15:30:09
```

---

## Rollback en vivo (sistema funcionando)

```bash
# Volver a la generación anterior
sudo nixos-rebuild switch --rollback

# Volver a una generación específica
sudo nix-env --switch-generation 45 --profile /nix/var/nix/profiles/system
sudo /nix/var/nix/profiles/system/bin/switch-to-configuration switch
```

---

## Rollback desde GRUB (sistema no bootea)

Al encender, en el menú de GRUB aparecen todas las generaciones disponibles. Si la última generación tiene un problema que impide bootear:

1. En GRUB, elegir "NixOS - All configurations"
2. Seleccionar la generación anterior
3. El sistema arranca con esa generación
4. Una vez adentro, corregir el problema en `configuration.nix`
5. `sudo nixos-rebuild switch` para crear una nueva generación correcta

---

## Rollback de Home Manager

```bash
# Ver generaciones de HM
home-manager generations

# Output:
# 2026-05-14 10:23 : id 12 -> /nix/store/xxx-home-manager-generation
# 2026-05-13 20:11 : id 11 -> /nix/store/yyy-home-manager-generation

# Activar una generación anterior
/nix/store/xxx-home-manager-generation/activate
```

---

## Limpiar generaciones viejas

```bash
# Ver cuánto ocupa
du -sh /nix/store

# Borrar generaciones de más de 30 días y hacer GC
sudo nix-collect-garbage --delete-older-than 30d
sudo nix store optimise   # deduplicar archivos idénticos

# Borrar TODAS las generaciones viejas (peligroso — sin rollback posible)
sudo nix-collect-garbage -d

# Configurar limpieza automática
# En configuration.nix:
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 30d";
};
```

> **Advertencia:** después de `nix-collect-garbage -d`, las generaciones viejas en GRUB ya no arrancan. Solo hacerlo cuando el sistema actual es estable.

---

## Snapshots btrfs como segunda capa de protección

Los rollbacks de NixOS protegen la configuración declarativa. Los snapshots btrfs protegen los datos (incluyendo la Nix Store y `/home`).

```bash
# Snapshot manual antes de cambio importante
sudo btrfs subvolume snapshot / /.snapshots/antes-del-cambio

# Con snapper (si está configurado)
sudo snapper -c root create --description "antes de actualizar nixpkgs"

# Revertir un snapshot btrfs (sistema no bootea, desde live USB)
# Montar el disco
mount /dev/nvme0n1p2 /mnt

# Ver snapshots disponibles
btrfs subvolume list /mnt

# Reemplazar @ con el snapshot
btrfs subvolume delete /mnt/@
btrfs subvolume snapshot /mnt/.snapshots/antes-del-cambio /mnt/@
```

Las dos capas de protección son independientes y complementarias:
- **Rollback NixOS:** cambia qué generación del sistema está activa (symlinks)
- **Snapshot btrfs:** restaura el estado real del disco (incluyendo cambios en /home)
