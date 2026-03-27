[[Systemd y Procesos]]

# Manejo y Eliminación de Procesos

## 1. Señales del Sistema

Las señales son mensajes que el kernel o un usuario envía a un proceso.

```bash
# Ver todas las señales disponibles
kill -l
```

**Señales más importantes:**

| Señal     | Número | Efecto                                                   |
| --------- | ------ | -------------------------------------------------------- |
| `SIGHUP`  | 1      | Recargar configuración (sin matar el proceso)            |
| `SIGINT`  | 2      | Interrumpir (equivale a Ctrl+C)                          |
| `SIGTERM` | 15     | Pedir al proceso que se cierre limpiamente (por defecto) |
| `SIGKILL` | 9      | Matar forzosamente (no se puede atrapar ni ignorar)      |
| `SIGSTOP` | 19     | Pausar proceso (equivale a Ctrl+Z)                       |
| `SIGCONT` | 18     | Reanudar proceso pausado                                 |

---

## 2. Envío de Señales

```bash
# Enviar SIGTERM (cierre limpio) por PID
kill PID

# Enviar una señal específica por PID
kill -9 PID            # SIGKILL (muerte forzada)
kill -HUP PID          # SIGHUP (recargar config)
kill -SIGHUP PID       # Equivalente

# Enviar señal a múltiples procesos
kill -SIGNAL PID1 PID2 PID3

# Enviar señal por nombre de proceso (no por PID)
pkill nombre_proceso
killall nombre_proceso
kill $(pidof nombre_proceso)

# Ejemplo: Recargar la config de SSHD sin matarlo
kill -HUP $(pidof sshd)
```

> **Regla:** Siempre intentar `kill PID` (SIGTERM) primero. Solo usar `kill -9` cuando el proceso no responde.

---

## 3. Jobs: Procesos en Primer y Segundo Plano

```bash
# Ejecutar un proceso en segundo plano
comando &
# Ejemplo: sleep 100 &

# Ver los jobs activos de esta terminal
jobs

# Pausar el proceso que está corriendo en primer plano
# Ctrl + Z

# Reanudar un job en primer plano
fg %job_id

# Reanudar un job en segundo plano
bg %job_id
```

---

## 4. Procesos Inmunes al Cierre de Terminal

```bash
# nohup: el proceso sigue corriendo aunque cierres la terminal
nohup comando &

# Ejemplo: descargar un archivo grande sin que se corte
nohup wget http://sitio.com/archivo.iso &
# La salida se guarda en nohup.out por defecto
```
