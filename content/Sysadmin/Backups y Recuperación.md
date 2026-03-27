[[0. General Tips]]

# Backups y Recuperación

## 1. La Regla 3-2-1

```
3 copias de tus datos
2 tipos de almacenamiento diferentes (disco local + nube, o disco + cinta)
1 copia fuera del sitio (offsite)
```

> Si no lo probaste restaurando, no es un backup. Es una esperanza.

---

## 2. rsync — Sincronización Inteligente

Solo copia lo que cambió. Mucho más eficiente que `cp` o `scp`.

```bash
# Sintaxis básica
rsync -avh /origen/ /destino/

# A un servidor remoto
rsync -avhz /datos/ usuario@IP:/backup/datos/

# Desde un servidor remoto
rsync -avhz usuario@IP:/datos/ /backup/local/

# Con puerto SSH personalizado
rsync -avhz -e "ssh -p 45678" /datos/ usuario@IP:/backup/

# Borrar en destino archivos que ya no existen en origen (mirror exacto)
rsync -avh --delete /origen/ /destino/

# Dry run (simular sin hacer nada)
rsync -avh --dry-run --delete /origen/ /destino/

# Excluir archivos o carpetas
rsync -avh --exclude='*.log' --exclude='.cache/' /origen/ /destino/
```

| Flag             | Significado                                      |
| ---------------- | ------------------------------------------------ |
| `-a`             | Archive (preserva permisos, fechas, links, etc.) |
| `-v`             | Verbose                                          |
| `-h`             | Tamaños legibles                                 |
| `-z`             | Comprimir durante la transferencia               |
| `--delete`       | Eliminar en destino lo que no existe en origen   |
| `--dry-run`      | Simular (no hacer cambios reales)                |
| `--progress`     | Mostrar progreso                                 |
| `--bwlimit=5000` | Limitar ancho de banda (KB/s)                    |

---

## 3. Script de Backup con rsync

```bash
#!/bin/bash
# /usr/local/bin/backup-diario.sh

# Variables
FECHA=$(date +%Y-%m-%d_%H%M)
ORIGEN="/var/www /etc /home"
DESTINO="/backup/diario"
LOG="/var/log/backup-diario.log"

echo "=== Backup iniciado: $FECHA ===" >> "$LOG"

for dir in $ORIGEN; do
    rsync -avh --delete "$dir" "$DESTINO" >> "$LOG" 2>&1
done

echo "=== Backup completado: $(date +%Y-%m-%d_%H%M) ===" >> "$LOG"

# Opcional: enviar notificación
# echo "Backup completado" | mail -s "Backup $FECHA" admin@ejemplo.com
```

```bash
# Hacerlo ejecutable
sudo chmod +x /usr/local/bin/backup-diario.sh

# Programarlo con cron (todos los días a las 2 AM)
sudo crontab -e
# 0 2 * * * /usr/local/bin/backup-diario.sh
```

---

## 4. Backups con Rotación (Mantener N copias)

```bash
#!/bin/bash
# /usr/local/bin/backup-rotacion.sh
# Mantiene los últimos 7 backups

FECHA=$(date +%Y-%m-%d)
ORIGEN="/var/www"
BASE_DESTINO="/backup"
DESTINO="$BASE_DESTINO/$FECHA"
RETENER=7

# Crear backup
rsync -avh --delete "$ORIGEN" "$DESTINO"

# Borrar backups más viejos que $RETENER días
find "$BASE_DESTINO" -maxdepth 1 -type d -mtime +$RETENER -exec rm -rf {} \;

echo "Backup $FECHA completado. Backups antiguos limpiados."
```

---

## 5. Backup de Base de Datos (PostgreSQL)

```bash
# Dump completo
sudo -u postgres pg_dump nombre_db > /backup/db/nombre_db_$(date +%Y%m%d).sql

# Dump comprimido
sudo -u postgres pg_dump nombre_db | gzip > /backup/db/nombre_db_$(date +%Y%m%d).sql.gz

# Dump de todas las bases
sudo -u postgres pg_dumpall > /backup/db/todas_$(date +%Y%m%d).sql

# Restaurar
sudo -u postgres psql nombre_db < /backup/db/nombre_db_20260325.sql

# Restaurar desde gzip
gunzip -c /backup/db/nombre_db_20260325.sql.gz | sudo -u postgres psql nombre_db
```

---

## 6. BorgBackup — Backups Deduplicados y Encriptados

Borg es la herramienta moderna para backups serios. Deduplica (no copia datos repetidos), comprime y encripta.

```bash
# Instalar
sudo apt install borgbackup

# Inicializar repositorio de backup (con encriptación)
borg init --encryption=repokey /backup/borg-repo

# Crear backup
borg create /backup/borg-repo::backup-{now:%Y-%m-%d} \
  /etc /home /var/www \
  --exclude '*.log' \
  --exclude '.cache'

# Listar backups existentes
borg list /backup/borg-repo

# Ver contenido de un backup
borg list /backup/borg-repo::backup-2026-03-25

# Restaurar un archivo específico
borg extract /backup/borg-repo::backup-2026-03-25 home/rolando/archivo.txt

# Restaurar todo
cd /
borg extract /backup/borg-repo::backup-2026-03-25

# Podar backups viejos (mantener últimos 7 diarios, 4 semanales, 6 mensuales)
borg prune /backup/borg-repo \
  --keep-daily=7 \
  --keep-weekly=4 \
  --keep-monthly=6

# Ver cuánto espacio ahorra la deduplicación
borg info /backup/borg-repo
```

### BorgBackup a servidor remoto

```bash
# El servidor remoto debe tener borg instalado
borg init --encryption=repokey ssh://usuario@IP-backup:45678/backup/borg-repo

borg create ssh://usuario@IP-backup:45678/backup/borg-repo::backup-{now:%Y-%m-%d} \
  /etc /home /var/www
```

---

## 7. Snapshots de LVM y Sistemas de Archivos

```bash
# Snapshot LVM (ver nota de Gestión de Discos)
sudo lvcreate -s -n snap-antes-update -L 5G /dev/vg/lv

# Si la actualización salió mal, restaurar
sudo lvconvert --merge /dev/vg/snap-antes-update
sudo reboot
```

---

## 8. Verificar Backups (Lo Más Importante)

```bash
# SIEMPRE probar que podés restaurar
# 1. Restaurar en un directorio temporal
mkdir /tmp/test-restore
cd /tmp/test-restore

# Para rsync:
rsync -avh /backup/diario/var/www/ /tmp/test-restore/

# Para borg:
borg extract /backup/borg-repo::backup-2026-03-25

# 2. Verificar que los archivos están completos
ls -la /tmp/test-restore/
diff -r /var/www/ /tmp/test-restore/www/

# 3. Limpiar
rm -rf /tmp/test-restore
```

> **Proceso recomendado:** Cada mes, hacer un restore de prueba. Documentar si funcionó o no. Si nunca probaste restaurar, no sabés si tu backup funciona.

---

## 9. Checklist de Backup para Servidores

```
[ ] /etc/ (configuración del sistema)
[ ] /home/ (datos de usuarios)
[ ] /var/www/ (sitios web)
[ ] /var/lib/postgresql/ (o dump de las bases)
[ ] /opt/ (software instalado manualmente)
[ ] Lista de paquetes instalados:
    dpkg --get-selections > /backup/paquetes.list
[ ] Crontabs:
    crontab -l > /backup/crontab-root.txt
[ ] Firewall:
    sudo ufw status > /backup/firewall-rules.txt
```
