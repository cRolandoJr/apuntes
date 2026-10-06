---
title: Nix para DevOps, en castellano
---

# Nix para DevOps, en castellano

Acá había una serie de apuntes sobre NixOS que escribí mientras migraba mi máquina desde Arch.
Los retiré en octubre de 2026: tenían errores y quedaron superados por un material nuevo, más
ordenado y verificado comando por comando.

## Qué viene

Una serie cronológica, en video y en texto, donde construyo en público una infraestructura
declarativa con NixOS, de la laptop a la nube:

- instalar un servidor desde la laptop con `nixos-anywhere` y `disko`;
- desplegar y hacer rollback sin entrar al servidor;
- escribir un módulo propio, manejar secretos con `sops-nix` y probar con tests de VM;
- CI con GitHub Actions, contenedores con `dockerTools`, Terraform/OpenTofu y un cluster de
  Kubernetes con nodos NixOS;
- observabilidad y, al final, criterio: dónde entra Nix y dónde no (Ansible, Terraform).

La idea que guía todo: *Terraform crea la infraestructura, NixOS define el host, y Ansible queda
para lo que no es NixOS.*

## Mientras tanto

- Mi configuración de NixOS es pública: <https://github.com/cRolandoJr/nix-config>
- Mis contribuciones a nixpkgs: <https://github.com/NixOS/nixpkgs/pulls/cRolandoJr>
- Comunidad en castellano: [NixOS Hispano](https://nixoshispano.org/)
