[[Automatización y Logs]]

# grep y find

## 1. grep — Buscar Texto Dentro de Archivos

Busca líneas que coincidan con un patrón dentro de archivos, sin necesidad de abrirlos.

### Sintaxis

```bash
grep [OPCIONES] patron archivo
```

### Opciones principales

| Opción        | Función                                                |
| ------------- | ------------------------------------------------------ |
| `-i`          | Ignorar mayúsculas/minúsculas                          |
| `-n`          | Mostrar número de línea                                |
| `-r` (o `-R`) | Buscar recursivamente en subcarpetas                   |
| `-v`          | Invertir la búsqueda (mostrar líneas que NO coinciden) |
| `-w`          | Buscar la palabra completa (no subcadenas)             |
| `-c`          | Mostrar solo la cantidad de coincidencias              |
| `-C n`        | Mostrar n líneas de contexto antes y después           |
| `-a`          | Buscar en archivos binarios                            |

### Ejemplos útiles

```bash
# Buscar "error" en syslog (sin importar mayúsculas)
grep -i "error" /var/log/syslog

# Buscar intentos de login fallidos en SSH
grep -i "failed" /var/log/auth.log

# Buscar recursivamente en una carpeta
grep -r "TODO" /home/rolando/proyecto/

# Buscar una palabra exacta con número de línea
grep -wn "root" /etc/passwd

# Contar cuántas veces aparece un patrón
grep -c "error" /var/log/syslog

# Mostrar 2 líneas de contexto antes y después
grep -C 2 "panic" /var/log/syslog
```

---

## 2. find — Buscar Archivos por Nombre, Tipo, Tamaño, etc.

Busca archivos en el sistema de archivos según criterios.

### Sintaxis

```bash
find RUTA [OPCIONES]
```

### Opciones principales

| Opción                | Función                                   |
| --------------------- | ----------------------------------------- |
| `-type f`             | Solo archivos                             |
| `-type d`             | Solo directorios                          |
| `-type l`             | Solo enlaces simbólicos                   |
| `-name "patron"`      | Buscar por nombre (sensible a mayúsculas) |
| `-iname "patron"`     | Buscar por nombre (ignora mayúsculas)     |
| `-size +1M`           | Archivos mayores a 1 MB                   |
| `-size -100k`         | Archivos menores a 100 KB                 |
| `-user usuario`       | Archivos del usuario                      |
| `-group grupo`        | Archivos del grupo                        |
| `-perm 755`           | Archivos con permisos específicos         |
| `-mtime -7`           | Modificados en los últimos 7 días         |
| `-exec comando {} \;` | Ejecutar un comando en cada resultado     |

### Ejemplos útiles

```bash
# Buscar todos los archivos .log en /var
find /var -type f -name "*.log"

# Buscar archivos mayores a 100 MB en todo el sistema
find / -type f -size +100M 2>/dev/null

# Buscar archivos modificados en las últimas 24 horas
find /home -type f -mtime -1

# Cambiar permisos de todos los archivos en home a 640
find ~ -type f -exec chmod 640 {} \;

# Buscar y borrar archivos .tmp (con confirmación)
find /tmp -type f -name "*.tmp" -exec rm -i {} \;
```

---

## 3. locate / plocate — Búsqueda Rápida por Índice

Más rápido que `find` porque busca en una base de datos indexada (no recorre el disco).

```bash
# Actualizar la base de datos (necesario después de crear archivos nuevos)
sudo updatedb

# Buscar archivo por nombre
locate archivo

# Búsqueda sin importar mayúsculas
locate -i archivo

# Buscar por nombre exacto (no parcial)
locate -r '/archivo$'

# Buscar solo por nombre base (sin ruta)
locate -b archivo

# Verificar que el archivo realmente existe (puede haber sido borrado)
locate -e archivo
```

---

## 4. which — Encontrar la Ruta de un Comando

```bash
# ¿Dónde está el ejecutable?
which python3

# Mostrar todas las ubicaciones si hay múltiples
which -a python
```
