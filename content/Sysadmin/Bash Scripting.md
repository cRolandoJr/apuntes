[[0. General Tips]]

# Bash Scripting

## 1. Lo Básico

```bash
#!/bin/bash
# Siempre iniciar con el shebang ^^

# Buenas prácticas obligatorias en scripts de producción:
set -euo pipefail
# -e = salir si un comando falla
# -u = error si se usa una variable no definida
# -o pipefail = fallar si cualquier comando en un pipe falla
```

---

## 2. Variables

```bash
# Asignación (SIN espacios alrededor del =)
NOMBRE="Rolando"
EDAD=26
FECHA=$(date +%Y-%m-%d)

# Uso (con comillas para evitar problemas con espacios)
echo "Hola $NOMBRE, hoy es $FECHA"
echo "Tu nombre tiene ${#NOMBRE} caracteres"

# Variables de entorno
export MI_VARIABLE="valor"

# Variables especiales
echo $0   # Nombre del script
echo $1   # Primer argumento
echo $2   # Segundo argumento
echo $#   # Cantidad de argumentos
echo $@   # Todos los argumentos (como lista)
echo $?   # Exit code del último comando
echo $$   # PID del script actual
echo $!   # PID del último proceso en background
```

### Arrays

```bash
# Declarar
FRUTAS=("manzana" "banana" "naranja")

# Acceder
echo "${FRUTAS[0]}"     # manzana
echo "${FRUTAS[@]}"     # todos los elementos
echo "${#FRUTAS[@]}"    # cantidad de elementos

# Agregar
FRUTAS+=("uva")

# Recorrer
for fruta in "${FRUTAS[@]}"; do
    echo "Fruta: $fruta"
done
```

---

## 3. Condicionales

```bash
# Estructura if/elif/else
if [[ condición ]]; then
    # acción
elif [[ otra_condición ]]; then
    # otra acción
else
    # acción por defecto
fi
```

### Comparaciones de Strings

```bash
[[ "$a" == "$b" ]]     # Iguales
[[ "$a" != "$b" ]]     # Distintos
[[ -z "$a" ]]          # String vacío
[[ -n "$a" ]]          # String no vacío
[[ "$a" == *.log ]]    # Pattern matching (wildcard)
[[ "$a" =~ ^[0-9]+$ ]] # Regex (solo números)
```

### Comparaciones Numéricas

```bash
[[ $a -eq $b ]]   # Igual
[[ $a -ne $b ]]   # Distinto
[[ $a -gt $b ]]   # Mayor que
[[ $a -ge $b ]]   # Mayor o igual
[[ $a -lt $b ]]   # Menor que
[[ $a -le $b ]]   # Menor o igual
```

### Tests de Archivos

```bash
[[ -f /ruta/archivo ]]   # Existe y es archivo
[[ -d /ruta/dir ]]       # Existe y es directorio
[[ -e /ruta ]]           # Existe (cualquier tipo)
[[ -r /ruta ]]           # Tiene permiso de lectura
[[ -w /ruta ]]           # Tiene permiso de escritura
[[ -x /ruta ]]           # Tiene permiso de ejecución
[[ -s /ruta ]]           # Existe y NO está vacío
[[ -L /ruta ]]           # Es symlink
```

### Operadores Lógicos

```bash
[[ cond1 && cond2 ]]     # AND
[[ cond1 || cond2 ]]     # OR
[[ ! condición ]]        # NOT
```

### Case (como switch)

```bash
case "$opcion" in
    start)
        echo "Iniciando..."
        ;;
    stop)
        echo "Deteniendo..."
        ;;
    restart)
        echo "Reiniciando..."
        ;;
    *)
        echo "Uso: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

---

## 4. Loops

```bash
# For clásico con lista
for i in 1 2 3 4 5; do
    echo "Número: $i"
done

# For con rango
for i in {1..10}; do
    echo "Número: $i"
done

# For estilo C
for ((i=0; i<10; i++)); do
    echo "Iteración: $i"
done

# For con archivos
for archivo in /var/log/*.log; do
    echo "Procesando: $archivo"
done

# While
contador=0
while [[ $contador -lt 5 ]]; do
    echo "Contador: $contador"
    ((contador++))
done

# Leer archivo línea por línea
while IFS= read -r linea; do
    echo "Línea: $linea"
done < /etc/passwd

# Until (hasta que sea verdadero)
until ping -c1 8.8.8.8 &>/dev/null; do
    echo "Esperando conexión..."
    sleep 5
done
echo "Conectado"
```

---

## 5. Funciones

```bash
# Declaración
verificar_root() {
    if [[ $EUID -ne 0 ]]; then
        echo "ERROR: Este script debe ejecutarse como root"
        exit 1
    fi
}

# Con argumentos
saludar() {
    local nombre="$1"    # 'local' limita el scope a la función
    local edad="${2:-desconocida}"   # Valor por defecto
    echo "Hola $nombre, tenés $edad años"
}

# Con return (solo valores numéricos 0-255)
es_par() {
    local num=$1
    return $(( num % 2 ))
}

# Uso
verificar_root
saludar "Rolando" 26

if es_par 4; then
    echo "4 es par"
fi

# Capturar salida de una función
obtener_ip() {
    hostname -I | awk '{print $1}'
}

MI_IP=$(obtener_ip)
echo "Mi IP es: $MI_IP"
```

---

## 6. Manejo de Errores y Exit Codes

```bash
# Exit codes: 0 = éxito, 1-255 = error
# Convenciones comunes:
# 0 = éxito
# 1 = error general
# 2 = uso incorrecto del comando
# 126 = permiso denegado
# 127 = comando no encontrado

# Manejar errores con || y &&
mkdir /tmp/test && echo "Creado" || echo "Falló"

# Función de error personalizada
error() {
    echo "[ERROR] $1" >&2   # >&2 escribe en stderr
    exit "${2:-1}"          # exit code por defecto: 1
}

# Usar
[[ -f /etc/config.conf ]] || error "Config no encontrado" 2

# Trap — ejecutar algo al salir (limpieza)
cleanup() {
    echo "Limpiando archivos temporales..."
    rm -f /tmp/mi_script_*.tmp
}
trap cleanup EXIT    # Se ejecuta siempre al salir del script
trap cleanup ERR     # Se ejecuta cuando hay error

# Trap para Ctrl+C
trap 'echo "Interrumpido"; exit 130' INT
```

---

## 7. Input del Usuario y Argumentos

```bash
# Leer input interactivo
read -p "¿Cómo te llamás? " nombre
read -sp "Contraseña: " password    # -s = silent (no muestra lo que escribís)
echo  # Salto de línea después del password

# Argumentos con getopts
while getopts "f:o:vh" opt; do
    case $opt in
        f) ARCHIVO="$OPTARG" ;;
        o) SALIDA="$OPTARG" ;;
        v) VERBOSE=true ;;
        h) echo "Uso: $0 -f archivo -o salida [-v]"; exit 0 ;;
        *) echo "Opción inválida"; exit 1 ;;
    esac
done

# Validar argumentos obligatorios
if [[ -z "${ARCHIVO:-}" ]]; then
    echo "ERROR: -f archivo es obligatorio"
    exit 1
fi
```

---

## 8. Redirección y Pipes

```bash
# Redirección básica
comando > archivo.txt      # Sobrescribir stdout
comando >> archivo.txt     # Agregar stdout
comando 2> errores.txt     # Solo stderr
comando &> todo.txt        # stdout + stderr
comando 2>&1               # Redirigir stderr a stdout

# /dev/null (descartar salida)
comando > /dev/null 2>&1   # Silenciar todo

# Here document (texto multilínea)
cat <<EOF > /etc/config.conf
servidor=localhost
puerto=8080
debug=false
EOF

# Here string
grep "error" <<< "$variable_con_texto"

# Process substitution
diff <(ls /dir1) <(ls /dir2)
```

---

## 9. Patrones Útiles

### Verificar si un comando existe

```bash
command -v nginx &>/dev/null || {
    echo "nginx no está instalado"
    exit 1
}
```

### Logging

```bash
LOG="/var/log/mi_script.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"
}

log "Script iniciado"
log "Procesando archivos..."
```

### Preguntar sí/no

```bash
confirmar() {
    read -p "$1 [s/N]: " respuesta
    [[ "$respuesta" =~ ^[sS]$ ]]
}

if confirmar "¿Reiniciar el servicio?"; then
    systemctl restart nginx
fi
```

### Esperar con timeout

```bash
esperar_servicio() {
    local intentos=30
    local contador=0
    while ! curl -sf http://localhost:8080/health &>/dev/null; do
        ((contador++))
        if [[ $contador -ge $intentos ]]; then
            echo "Timeout esperando el servicio"
            return 1
        fi
        sleep 1
    done
    echo "Servicio disponible"
}
```

---

## 10. Debugging

```bash
# Ejecutar script con debug
bash -x mi_script.sh

# Activar debug dentro del script (en una sección)
set -x    # Activar debug
# ... código a debuggear ...
set +x    # Desactivar debug

# Ver qué haría el script sin ejecutar (dry-run)
# No existe nativo en bash, pero podés implementarlo:
DRY_RUN=${DRY_RUN:-false}

ejecutar() {
    if [[ "$DRY_RUN" == "true" ]]; then
        echo "[DRY-RUN] $*"
    else
        "$@"
    fi
}

ejecutar systemctl restart nginx
# Uso: DRY_RUN=true ./mi_script.sh
```

---

## 11. Template de Script de Producción

```bash
#!/bin/bash
set -euo pipefail

# =====================================================
# Descripción: Qué hace este script
# Autor: Rolando
# Uso: ./script.sh -f archivo [-v]
# =====================================================

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG="/var/log/$(basename "$0" .sh).log"

# --- Funciones ---
log()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"; }
error() { echo "[ERROR] $1" >&2; exit "${2:-1}"; }

cleanup() {
    log "Limpieza ejecutada"
}
trap cleanup EXIT

verificar_root() {
    [[ $EUID -eq 0 ]] || error "Ejecutar como root"
}

# --- Main ---
main() {
    verificar_root
    log "Script iniciado"

    # Tu lógica acá

    log "Script completado exitosamente"
}

main "$@"
```
