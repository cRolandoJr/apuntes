[[0. General Tips]]

# Discos, Memoria RAM y Archivos

## 1. Espacio en Disco

```bash
# Ver espacio libre por partición
df -h

# Ver TOP 5 carpetas más pesadas en el directorio actual
du -sh * | sort -rh | head -5

# Herramienta visual interactiva (instalar: sudo apt install ncdu)
ncdu /

# Listar discos y particiones del sistema
lsblk

# Ver información detallada de particiones
sudo fdisk -l

# Ver puntos de montaje activos
mount | column -t
```

---

## 2. Memoria RAM

```bash
# Ver uso de RAM
free -h
```

**Cómo leer la salida de `free -h`:**

| Columna        | Significado                                                         |
| -------------- | ------------------------------------------------------------------- |
| **total**      | RAM física instalada                                                |
| **used**       | RAM en uso por procesos                                             |
| **free**       | RAM completamente libre (sin usar)                                  |
| **buff/cache** | RAM usada para caché de disco (se libera si un proceso la necesita) |
| **available**  | RAM real disponible para nuevas aplicaciones                        |

> **Concepto clave:** Linux usa la RAM libre como caché de disco para acelerar lecturas. Si una app necesita más RAM, el kernel libera caché automáticamente. **"Free RAM is wasted RAM"** — lo importante es la columna `available`, no `free`.

```bash
# Ver información de las memorias físicas instaladas (módulos RAM)
sudo dmidecode -t memory
```

---

## 3. Compresión

Ver nota completa: [[Tar]]

---

## 4. Copiar Archivos entre Servidores (SCP)

```bash
# Sintaxis: scp origen destino
# Copiar archivo DESDE un servidor remoto a tu máquina
scp usuario@IP:/ruta/archivo /destino/local

# Copiar archivo HACIA un servidor remoto
scp /archivo/local usuario@IP:/ruta/destino

# Copiar una carpeta completa (-r = recursivo)
scp -r usuario@IP:/ruta/carpeta /destino/local

# Si cambiaste el puerto SSH (ej: 45678)
scp -P 45678 usuario@IP:/ruta/archivo /destino/local
```

**Ejemplo real:**

```bash
scp rolando@192.168.122.50:~/tz.txt clases/AdminSist/
```
