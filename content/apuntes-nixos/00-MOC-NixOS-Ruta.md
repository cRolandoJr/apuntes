# NixOS — Ruta de Aprendizaje

Migración de Arch Linux a NixOS en Victus 15fb2xxx (AMD, 32 GB RAM, WD SN850x).
Destino final: NixOS declarativo con Flakes + Home Manager + Hyprland + btrfs.

---

## Etapa 0 — Fundamentos

- [[01-Que-es-Nix-y-Por-que]] — El problema que resuelve, inmutabilidad, la Nix Store
- [[02-Lenguaje-Nix]] — Tipos, funciones, let/in, with, inherit, import

## Etapa 1 — Instalación

- [[03-Anatomia-configuration-nix]] — Qué es cada sección y cómo se relacionan
- [[04-Btrfs-y-Particionado]] — Subvolúmenes, opciones de montaje, layout recomendado
- [[05-Instalacion-limpia]] — Paso a paso desde la ISO hasta el primer boot

## Etapa 2 — Gestión de Paquetes

- [[06-Gestion-de-Paquetes]] — nix-env vs nix profile vs declarativo
- [[07-Canales-Flakes-nixpkgs]] — Diferencias, por qué Flakes es el futuro, primeros pasos

## Etapa 3 — Home Manager e Hyprland

- [[08-Home-Manager]] — Qué es, cómo se integra, estructura básica
- [[09-Migracion-Hyprland]] — Migrar la config existente de Arch a Home Manager

## Etapa 4 — Avanzado

- [[10-Overlays-y-Overrides]] — Parchear y personalizar paquetes
- [[11-Rollbacks-y-Generaciones]] — Volver atrás si algo falla
- [[12-Debugging]] — nix log, nix why-depends, nix repl, errores comunes

---

## Hardware de referencia

| Componente | Detalle |
|------------|---------|
| Laptop | Victus 15fb2xxx |
| CPU | AMD |
| RAM | 32 GB |
| SSD destino | WD SN850x (nuevo, instalación limpia) |
| Filesystem | btrfs (sin /home separado) |
| DE inicio | Plasma (ISO) → Hyprland (Home Manager) |
