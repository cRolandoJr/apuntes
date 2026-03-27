[[0. General Tips]]

# sed, awk y Expresiones Regulares

## 1. Expresiones Regulares (Regex)

Las regex son el lenguaje universal para buscar y manipular texto. Se usan en grep, sed, awk, y casi cualquier herramienta.

### Caracteres Básicos

| Patrón | Significado               | Ejemplo                                    |
| ------ | ------------------------- | ------------------------------------------ |
| `.`    | Cualquier carácter        | `a.c` → abc, a1c, a-c                      |
| `*`    | 0 o más del anterior      | `ab*c` → ac, abc, abbc                     |
| `+`    | 1 o más del anterior      | `ab+c` → abc, abbc (no ac)                 |
| `?`    | 0 o 1 del anterior        | `ab?c` → ac, abc                           |
| `^`    | Inicio de línea           | `^Error` → líneas que empiezan con "Error" |
| `$`    | Fin de línea              | `log$` → líneas que terminan en "log"      |
| `\`    | Escapar carácter especial | `\.` → un punto literal                    |

### Clases de Caracteres

| Patrón        | Significado                  |
| ------------- | ---------------------------- |
| `[abc]`       | a, b, o c                    |
| `[a-z]`       | Cualquier minúscula          |
| `[A-Z]`       | Cualquier mayúscula          |
| `[0-9]`       | Cualquier dígito             |
| `[^abc]`      | Cualquier cosa MENOS a, b, c |
| `[a-zA-Z0-9]` | Alfanumérico                 |

### Clases POSIX (para usar en grep/sed)

| Clase       | Equivale a               |
| ----------- | ------------------------ |
| `[:digit:]` | `[0-9]`                  |
| `[:alpha:]` | `[a-zA-Z]`               |
| `[:alnum:]` | `[a-zA-Z0-9]`            |
| `[:space:]` | Espacios, tabs, newlines |
| `[:upper:]` | `[A-Z]`                  |
| `[:lower:]` | `[a-z]`                  |

### Cuantificadores

| Patrón  | Significado         |
| ------- | ------------------- |
| `{3}`   | Exactamente 3 veces |
| `{2,5}` | Entre 2 y 5 veces   |
| `{3,}`  | 3 o más veces       |

### Grupos y Alternancia

```
(abc)     → Grupo (captura "abc" junto)
(a|b)     → a O b
\1        → Referencia al primer grupo capturado
```

### Ejemplos Prácticos

```bash
# IP address (simplificado)
[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+

# Email (simplificado)
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}

# Fecha YYYY-MM-DD
[0-9]{4}-[0-9]{2}-[0-9]{2}

# Líneas vacías
^$

# Líneas que NO son comentarios
^[^#]
```

---

## 2. sed — Stream Editor

sed procesa texto línea por línea. Ideal para buscar y reemplazar.

### Sustitución (lo más usado)

```bash
# Reemplazar primera ocurrencia por línea
sed 's/viejo/nuevo/' archivo.txt

# Reemplazar TODAS las ocurrencias por línea (flag g)
sed 's/viejo/nuevo/g' archivo.txt

# Ignorar mayúsculas/minúsculas (flag I)
sed 's/error/ERROR/gI' archivo.txt

# Editar el archivo IN-PLACE (modificar directamente)
sed -i 's/viejo/nuevo/g' archivo.txt

# In-place con backup
sed -i.bak 's/viejo/nuevo/g' archivo.txt

# Usar otro delimitador (útil con rutas)
sed 's|/var/www/old|/var/www/new|g' archivo.txt

# Solo en líneas que contengan un patrón
sed '/ERROR/s/viejo/nuevo/g' archivo.txt

# Solo en líneas 5 a 10
sed '5,10s/viejo/nuevo/g' archivo.txt
```

### Eliminar Líneas

```bash
# Borrar línea 3
sed '3d' archivo.txt

# Borrar líneas 5 a 10
sed '5,10d' archivo.txt

# Borrar líneas que contengan "DEBUG"
sed '/DEBUG/d' archivo.txt

# Borrar líneas vacías
sed '/^$/d' archivo.txt

# Borrar comentarios (líneas que empiezan con #)
sed '/^#/d' archivo.txt

# Borrar comentarios Y líneas vacías
sed '/^#/d; /^$/d' archivo.txt
```

### Insertar y Agregar

```bash
# Insertar línea ANTES de la línea 3
sed '3i\Línea nueva insertada' archivo.txt

# Agregar línea DESPUÉS de la línea 3
sed '3a\Línea nueva agregada' archivo.txt

# Agregar después de líneas que contengan "server"
sed '/server/a\    nueva_directiva;' archivo.txt
```

### Imprimir

```bash
# Imprimir solo líneas que matcheen (como grep)
sed -n '/ERROR/p' archivo.txt

# Imprimir líneas 10 a 20
sed -n '10,20p' archivo.txt

# Imprimir número de línea + contenido
sed -n '/ERROR/=' archivo.txt
```

### Ejemplos Prácticos

```bash
# Cambiar puerto en un config
sed -i 's/^port=8080/port=9090/' config.conf

# Descomentar una línea
sed -i 's/^#ListenAddress 0.0.0.0/ListenAddress 0.0.0.0/' /etc/ssh/sshd_config

# Comentar una línea
sed -i 's/^ListenAddress/#ListenAddress/' /etc/ssh/sshd_config

# Agregar texto al final de líneas que matcheen
sed -i '/^server_name/s/$/;/' nginx.conf

# Reemplazar usando grupos capturados
echo "2026-03-25" | sed 's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)/\3\/\2\/\1/'
# Resultado: 25/03/2026
```

---

## 3. awk — Procesador de Texto por Campos

awk divide cada línea en campos (por defecto separados por espacios/tabs).

### Básico

```bash
# Imprimir campo 1 (primera "columna")
awk '{print $1}' archivo.txt

# Imprimir campos 1 y 3
awk '{print $1, $3}' archivo.txt

# Imprimir toda la línea
awk '{print $0}' archivo.txt

# Imprimir último campo
awk '{print $NF}' archivo.txt

# Cambiar separador de campos
awk -F: '{print $1, $3}' /etc/passwd
# Resultado: usuario UID

# Separador de salida
awk -F: 'BEGIN{OFS=","} {print $1, $3, $6}' /etc/passwd
# Resultado: usuario,UID,home
```

### Variables Built-in

| Variable    | Significado                               |
| ----------- | ----------------------------------------- |
| `$0`        | Línea completa                            |
| `$1, $2...` | Campo 1, 2, etc.                          |
| `NF`        | Número de campos en la línea actual       |
| `NR`        | Número de línea actual (registro)         |
| `FS`        | Separador de campos de entrada            |
| `OFS`       | Separador de campos de salida             |
| `RS`        | Separador de registros (default: newline) |

### Filtros y Condiciones

```bash
# Solo líneas que contengan "error"
awk '/error/' archivo.txt

# Solo líneas donde el campo 3 sea mayor a 100
awk '$3 > 100' archivo.txt

# Solo si campo 1 es "root"
awk -F: '$1 == "root"' /etc/passwd

# Combinar condiciones
awk -F: '$3 >= 1000 && $7 != "/usr/sbin/nologin"' /etc/passwd

# Imprimir número de línea y contenido
awk '{print NR": "$0}' archivo.txt
```

### Bloques BEGIN y END

```bash
# BEGIN se ejecuta antes de procesar, END después
awk 'BEGIN{print "=== REPORTE ==="} {print $0} END{print "Total líneas: "NR}' archivo.txt

# Sumar valores de una columna
awk '{suma += $3} END{print "Total:", suma}' datos.txt

# Promedio
awk '{suma += $1; n++} END{print "Promedio:", suma/n}' numeros.txt

# Contar ocurrencias
awk '/ERROR/{count++} END{print "Errores:", count}' log.txt
```

### Printf (formato controlado)

```bash
# Formato tipo C
awk -F: '{printf "%-20s UID: %5d\n", $1, $3}' /etc/passwd
# Resultado:
# root                 UID:     0
# rolando              UID:  1000
```

---

## 4. Combinaciones Potentes

```bash
# Top 10 IPs en access.log de nginx
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Uso de disco por directorio, ordenado
du -sh /var/*/ 2>/dev/null | sort -rh | head -10

# Encontrar procesos que usan más de 10% CPU
ps aux | awk '$3 > 10.0 {print $1, $2, $3, $11}'

# Extraer solo las IPs de la config de red
ip addr | awk '/inet / {print $2}'

# Usuarios con shell de login (no nologin/false)
awk -F: '$7 !~ /(nologin|false)$/ {print $1, $7}' /etc/passwd

# Tamaño total de archivos .log
find /var/log -name "*.log" -exec du -b {} \; | awk '{total += $1} END{printf "Total: %.2f MB\n", total/1024/1024}'

# Reemplazar en múltiples archivos
find /var/www -name "*.conf" -exec sed -i 's/viejo_dominio/nuevo_dominio/g' {} \;

# Extraer errores 5xx del log con timestamp
awk '$9 >= 500 && $9 < 600 {print $4, $7, $9}' /var/log/nginx/access.log

# Monitorear log en tiempo real filtrando errores
tail -f /var/log/syslog | awk '/error|fail|critical/ {print strftime("%H:%M:%S"), $0}'
```

---

## 5. Cheat Sheet Rápido

```
# sed
sed 's/a/b/g'        Reemplazar todo
sed -i                Editar in-place
sed '/patron/d'       Borrar líneas
sed -n '/patron/p'    Imprimir solo matches

# awk
awk '{print $1}'      Primer campo
awk -F:               Separador ":"
awk '/patron/'        Filtrar
awk '{sum+=$1} END{}' Sumar columna

# Regex
.    cualquier carácter
*    0 o más
+    1 o más
^    inicio de línea
$    fin de línea
[]   clase de caracteres
()   grupo
|    OR
```
