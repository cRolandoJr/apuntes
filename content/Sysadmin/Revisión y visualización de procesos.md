[[Systemd y Procesos]]

# Revisión y Visualización de Procesos

## 1. `htop` — Monitor Interactivo (Recomendado)

Mejor que `top`. Muestra uso de CPU, RAM y procesos en tiempo real con colores e interacción.

```bash
htop
```

### Atajos de teclado

| Tecla       | Función                                      |
| ----------- | -------------------------------------------- |
| Flechas     | Navegar entre procesos                       |
| `F1` / `h`  | Ayuda                                        |
| `F2`        | Configuración                                |
| `F3`        | Buscar proceso por nombre                    |
| `F4`        | Filtrar procesos                             |
| `F5` / `t`  | Vista de árbol (jerarquía padre-hijo)        |
| `F6` / `>`  | Ordenar por columna (CPU, MEM, etc.)         |
| `F7` / `]`  | Disminuir prioridad (nice) del proceso       |
| `F8` / `[`  | Aumentar prioridad (nice) del proceso        |
| `F9` / `k`  | Enviar señal (matar) al proceso seleccionado |
| `F10` / `q` | Salir                                        |

---

## 2. `top` — Monitor Clásico

```bash
top
```

### Atajos dentro de top

| Tecla     | Función                                              |
| --------- | ---------------------------------------------------- |
| `h`       | Ayuda                                                |
| `espacio` | Refrescar manualmente                                |
| `d`       | Cambiar intervalo de refresco (en segundos)          |
| `q`       | Salir                                                |
| `u`       | Filtrar por usuario                                  |
| `m`       | Cambiar visualización de memoria                     |
| `1`       | Ver estadísticas por cada CPU                        |
| `x` / `y` | Resaltar proceso activo y columna de ordenamiento    |
| `b`       | Alternar entre resaltado bold y texto                |
| `<` / `>` | Mover columna de ordenamiento izquierda/derecha      |
| `F`       | Entrar a gestión de campos (agregar/quitar columnas) |
| `W`       | Guardar configuración actual                         |

```bash
# Ejecutar top en modo batch (para guardar salida a un archivo)
# 3 refrescos, 1 segundo de intervalo
top -d 1 -n 3 -b > top_procesos.txt
```

---

## 3. `ps` — Snapshot de Procesos

```bash
# Procesos del terminal actual
ps

# Todos los procesos del sistema (formato completo)
ps -ef
ps aux

# Todos los procesos ordenados por uso de memoria
ps aux --sort=%mem | less

# Todos los procesos ordenados por uso de CPU
ps aux --sort=%cpu | less

# Vista de árbol (muestra jerarquía padre-hijo)
ps -ef --forest

# Procesos de un usuario específico
ps -f -u nombre_usuario
```

---

## 4. Otros Comandos Útiles

```bash
# Verificar si un comando es built-in del shell o un ejecutable
type rm         # => rm is /usr/bin/rm
type cd         # => cd is a shell built-in

# Verificar si un proceso específico está corriendo
pgrep -l sshd
ps -ef | grep sshd

# Ver árbol jerárquico de todos los procesos
pstree

# Sin agrupar ramas idénticas
pstree -c
```
