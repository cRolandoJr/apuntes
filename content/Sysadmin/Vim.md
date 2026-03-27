[[0. General Tips]]

# Vim — Editor de Texto en Terminal

## Modos de Operación

Vim tiene 3 modos principales:

| Modo                  | Propósito                         | Cómo entrar              |
| --------------------- | --------------------------------- | ------------------------ |
| **Comando**           | Navegar, copiar, borrar, buscar   | `ESC` (modo por defecto) |
| **Inserción**         | Escribir texto                    | `i`, `a`, `o`, etc.      |
| **Línea de comandos** | Guardar, salir, buscar/reemplazar | `:`                      |

> Archivo de configuración: `~/.vimrc`

---

## Entrar a Modo Inserción

| Tecla | Acción                             |
| ----- | ---------------------------------- |
| `i`   | Insertar antes del cursor          |
| `I`   | Insertar al inicio de la línea     |
| `a`   | Insertar después del cursor        |
| `A`   | Insertar al final de la línea      |
| `o`   | Insertar en una nueva línea abajo  |
| `O`   | Insertar en una nueva línea arriba |

---

## Modo Comando — Navegación y Edición

### Movimiento

| Tecla            | Acción                            |
| ---------------- | --------------------------------- |
| `h j k l`        | Izquierda, abajo, arriba, derecha |
| `0` o `^`        | Inicio de línea                   |
| `$`              | Final de línea                    |
| `G`              | Ir al final del archivo           |
| `gg`             | Ir al inicio del archivo          |
| `:n` (ej: `:10`) | Ir a la línea n                   |
| `w`              | Saltar a la siguiente palabra     |
| `b`              | Saltar a la palabra anterior      |

### Edición

| Tecla     | Acción                                   |
| --------- | ---------------------------------------- |
| `x`       | Borrar carácter bajo el cursor           |
| `dd`      | Cortar línea actual                      |
| `5dd`     | Cortar 5 líneas                          |
| `yy`      | Copiar línea actual                      |
| `p`       | Pegar después del cursor                 |
| `P`       | Pegar antes del cursor                   |
| `u`       | Deshacer                                 |
| `Ctrl+r`  | Rehacer                                  |
| `Shift+v` | Seleccionar línea completa (modo visual) |

### Búsqueda

| Tecla    | Acción                 |
| -------- | ---------------------- |
| `/texto` | Buscar hacia adelante  |
| `?texto` | Buscar hacia atrás     |
| `n`      | Siguiente coincidencia |
| `N`      | Coincidencia anterior  |

---

## Línea de Comandos (`:` desde modo comando)

| Comando             | Acción                                 |
| ------------------- | -------------------------------------- |
| `:w`                | Guardar                                |
| `:q`                | Salir                                  |
| `:wq` o `ZZ`        | Guardar y salir                        |
| `:q!`               | Salir sin guardar                      |
| `:e!`               | Volver a la última versión guardada    |
| `:set nu`           | Mostrar números de línea               |
| `:set nonu`         | Ocultar números de línea               |
| `:syntax on`        | Activar colores de sintaxis            |
| `:%s/viejo/nuevo/g` | Buscar y reemplazar en todo el archivo |

---

## Múltiples Archivos

```bash
# Abrir archivos apilados (ventanas horizontales)
vim -o archivo1 archivo2

# Abrir archivos y resaltar diferencias (como diff)
vim -d archivo1 archivo2

# Moverse entre ventanas
Ctrl+w luego flecha
```
