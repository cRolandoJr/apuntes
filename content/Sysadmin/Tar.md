[[Discos, Memoria Ram y Archivos]]

# Tar — Empaquetado y Compresión

## Mnemotecnia: **Z**ip **C**reate **V**erbose **F**ile (`zcvf`)

> **Sintaxis:** `tar -flags DESTINO ORIGEN` — "Quiero crear ESTO a partir de AQUELLO".

## Opciones Principales

| Opción | Función                                           |
| ------ | ------------------------------------------------- |
| `-c`   | **C**rear un nuevo archivo                        |
| `-x`   | **E**xtraer archivos                              |
| `-v`   | **V**erbose (mostrar detalles)                    |
| `-f`   | Especifica el **f**ichero a crear o leer          |
| `-t`   | Lis**t**ar el contenido sin extraer               |
| `-z`   | Comprimir/Descomprimir con **Gzip** (`.tar.gz`)   |
| `-j`   | Comprimir/Descomprimir con **Bzip2** (`.tar.bz2`) |
| `-J`   | Comprimir/Descomprimir con **XZ** (`.tar.xz`)     |

## Ejemplos

```bash
# Crear y comprimir con Gzip
tar -zcvf web-backup.tar.gz /var/www/html

# Crear y comprimir con Bzip2
tar -cjvf archivo.tar.bz2 /ruta/a/directorios

# Crear y comprimir con XZ (mejor compresión, más lento)
tar -cJvf archivo.tar.xz /ruta/a/directorios

# Listar contenido sin extraer
tar -tvf archivo.tar.gz

# Descomprimir
tar -xvf archivo.tar.gz

# Descomprimir en un directorio específico
tar -xvf archivo.tar.gz -C /ruta/destino

# Excluir archivos al comprimir
tar -zcvf backup.tar.gz /datos --exclude='*.log' --exclude='*.tmp'
```
