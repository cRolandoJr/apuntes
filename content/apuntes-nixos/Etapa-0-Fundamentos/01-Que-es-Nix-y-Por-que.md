# Qué es Nix y por qué existe

## El problema que Nix resuelve

Para entender Nix, primero hay que entender qué está roto en los gestores de paquetes tradicionales. No es que pacman o apt sean malos — es que tienen un límite estructural.

### El modelo tradicional: un único estado global

En Arch, cuando instalás un paquete:

```bash
pacman -S python
```

Pasa esto:
1. Se copian archivos a `/usr/lib/python3.x/`, `/usr/bin/python`, etc.
2. Se modifica el estado global del sistema
3. No hay registro de qué archivos pertenecen a qué versión exacta

Ese "estado global" es el problema:

- **No podés tener dos versiones del mismo paquete instaladas simultáneamente** (salvo hacks)
- **Actualizar puede romper otras cosas** — si `python 3.11 → 3.12` cambia una API, todo lo que depende de ella se rompe al mismo tiempo, sin aviso
- **"Funciona en mi máquina"** — tu entorno acumuló años de cambios manuales que no están documentados en ningún lado. Si perdés el disco, no podés reproducirlo exactamente
- **No podés deshacer** — `pacman -Rns` intenta limpiar, pero si algo quedó mal después de un update, no hay forma de volver atrás de forma garantizada

### El modelo Nix: inmutabilidad y aislamiento

Nix parte de una premisa distinta:

> **Un paquete es una función pura.** Las mismas entradas siempre producen la misma salida. Y esa salida nunca se modifica.

En la práctica, cada paquete vive en una ruta única basada en un hash criptográfico de todo lo que lo define:

```
/nix/store/ybhlx6rqv16pv5sl7998x0gf1gl4b4nm-python3-3.12.3/
            ↑ hash de: fuentes + dependencias + flags de compilación
```

Ese directorio **nunca se modifica**. Es de solo lectura. Siempre.

Resultado: múltiples versiones coexisten sin conflicto:

```
/nix/store/ybhlx6rqv16pv5sl7998x0gf1gl4b4nm-python3-3.12.3/
/nix/store/7v1k9...anterior-python3-3.11.9/
```

---

## La Nix Store — el corazón de todo

`/nix/store/` es el único lugar donde viven los paquetes en NixOS. Todo lo demás son symlinks.

```
/nix/store/
├── abc123...-firefox-124/
│   ├── bin/firefox
│   └── lib/...
├── def456...-python3-3.12/
│   └── bin/python3
└── ghi789...-python3-3.11/    ← otra versión, sin problema
    └── bin/python3
```

Cuando "instalás" algo, Nix agrega un symlink en tu perfil que apunta a ese directorio. Cuando "desinstalás", borra el symlink. El directorio en la store queda intacto hasta que corra el garbage collector.

---

## Declarativo vs imperativo

| Modelo | Cómo funciona | Ejemplo |
|--------|--------------|---------|
| **Imperativo** (pacman/apt) | Le decís qué hacer paso a paso. El estado es el resultado de todos los comandos de la historia | `pacman -S neovim`, `pacman -R vim`... |
| **Declarativo** (Nix) | Le decís cómo debe quedar el sistema. Nix calcula qué hacer para llegar ahí | `environment.systemPackages = [ pkgs.neovim ]` |

**Imperativo:** El sistema es el resultado de una secuencia de comandos. Para saber cómo está, tenés que conocer toda la historia.

**Declarativo:** El sistema *es* el archivo de configuración. Si perdés el disco, copiás `configuration.nix` a una instalación nueva y queda idéntico.

---

## Las tres cosas que es "Nix"

| Nombre | Qué es | Para qué |
|--------|--------|----------|
| **Nix** (el lenguaje) | Lenguaje de programación funcional y lazy | Describir paquetes y configuraciones |
| **Nix** (el gestor de paquetes) | Herramienta CLI | Instalar paquetes, gestionar la store |
| **NixOS** | Distribución Linux | Todo el sistema configurado declarativamente |

Podés usar el gestor Nix en Arch, macOS, o cualquier Linux sin instalar NixOS.

---

## Por qué btrfs + NixOS es una combinación natural

| Característica | Beneficio para NixOS |
|----------------|---------------------|
| **Copy-on-Write (CoW)** | Nuevas generaciones comparten bloques idénticos con las anteriores — menos espacio real del que parece |
| **Snapshots instantáneos** | Antes de cada `nixos-rebuild`, podés snapshotear. Si falla el boot, revertís desde GRUB sin depender de Nix |
| **Compresión zstd** | ~30-40% menos uso de disco sin costo perceptible en CPU moderno |

---

## El ciclo de vida en NixOS

```
1. Editás configuration.nix
      ↓
2. nixos-rebuild switch
      ↓
3. Nix evalúa la configuración (lenguaje Nix)
      ↓
4. Calcula qué paquetes faltan → los descarga/compila a /nix/store/
      ↓
5. Crea una nueva "generación" (symlinks actualizados)
      ↓
6. Activa la nueva generación (sin reiniciar para la mayoría de cambios)
      ↓
7. La generación anterior sigue existiendo → nixos-rebuild switch --rollback
```

Cada generación aparece en el menú de GRUB al bootear.

---

## Qué NO resuelve Nix (honestidad)

- **Curva de aprendizaje:** el lenguaje funcional y lazy es diferente a lo que estás acostumbrado
- **Errores crípticos:** los mensajes de error pueden ser difíciles de leer al principio
- **Software propietario:** algunos programas son más difíciles de empaquetar
- **Build times:** si un paquete no está en el cache binario, Nix lo compila desde fuente
