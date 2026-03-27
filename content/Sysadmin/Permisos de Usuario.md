[[Administración de cuentas y usuarios]]

# Permisos de Usuario

## 1. Cómo Leer los Permisos

El formato es: `Dueño | Grupo | Otros`

Cada bloque tiene 3 permisos: **r** (leer), **w** (escribir), **x** (ejecutar).

```
-rwxr-xr--
│└──┴──┴──
│  │   │   └─ Otros: r-- (solo leer)
│  │   └───── Grupo: r-x (leer y ejecutar)
│  └───────── Dueño: rwx (todo)
└──────────── Tipo: - (archivo), d (directorio), l (enlace)
```

## 2. Calculadora Rápida (Sistema Octal)

| Número | Permiso      |
| ------ | ------------ |
| **4**  | Leer (r)     |
| **2**  | Escribir (w) |
| **1**  | Ejecutar (x) |
| **0**  | Nada         |

Se suman para combinar:

- `7` = 4+2+1 = rwx (todo)
- `6` = 4+2 = rw- (leer y escribir)
- `5` = 4+1 = r-x (leer y ejecutar)
- `4` = solo leer

**Ejemplo:** `755` = Dueño(7=rwx), Grupo(5=r-x), Otros(5=r-x)

---

## 3. Comandos de Permisos

```bash
# Cambiar permisos (modo octal)
chmod 755 archivo
chmod 640 archivo

# Cambiar permisos (modo simbólico)
chmod +x script.sh              # Agregar ejecución para todos
chmod u+x script.sh              # Agregar ejecución solo al dueño
chmod go-w archivo               # Quitar escritura a grupo y otros

# Cambiar dueño y grupo
sudo chown usuario:grupo archivo
sudo chown -R usuario:grupo carpeta/   # Recursivo

# Cambiar solo el grupo
sudo chgrp grupo archivo

# Cambiar permisos de todos los archivos en home
find ~ -type f -exec chmod 640 {} \;

# Cambiar permisos de todos los directorios en home
find ~ -type d -exec chmod 750 {} \;
```

---

## 4. Permisos Críticos para SSH

SSH es estricto con permisos. Si están mal, **rechaza la conexión sin decirte por qué:**

| Elemento                         | Permiso Requerido    |
| -------------------------------- | -------------------- |
| Carpeta `~/.ssh/`                | `700` (`drwx------`) |
| `authorized_keys`                | `600` (`-rw-------`) |
| Llave privada (`id_ed25519`)     | `600` (`-rw-------`) |
| Llave pública (`id_ed25519.pub`) | `644` (`-rw-r--r--`) |

---

## 5. Permisos Especiales

### SUID (Set User ID)

Cuando se ejecuta, el proceso corre con los permisos del **dueño del archivo**, no del usuario que lo ejecutó.

```bash
# Ver: la "s" reemplaza la "x" del dueño
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
# Cualquier usuario puede ejecutar passwd, y corre como root

# Establecer SUID
chmod u+s archivo
chmod 4755 archivo   # El 4 al principio = SUID

# Buscar archivos con SUID en el sistema (auditoría de seguridad)
find / -type f -perm -4000 2>/dev/null
```

### SGID (Set Group ID)

El proceso corre con el grupo del archivo. En directorios, los archivos nuevos heredan el grupo.

```bash
# Establecer SGID
chmod g+s directorio/
chmod 2755 directorio/   # El 2 al principio = SGID
```

### Sticky Bit

En un directorio, solo el dueño de un archivo puede borrarlo (aunque otros tengan permiso de escritura). Ejemplo clásico: `/tmp`.

```bash
# Ver: la "t" al final
ls -ld /tmp
# drwxrwxrwt ... /tmp

# Establecer sticky bit
chmod +t directorio/
chmod 1755 directorio/   # El 1 al principio = Sticky
```

---

## 6. umask — Permisos por Defecto

Define qué permisos se **quitan** al crear archivos y directorios nuevos.

```bash
# Ver umask actual
umask
# Resultado típico: 0022

# Significa:
# Archivos nuevos: 666 - 022 = 644 (rw-r--r--)
# Directorios nuevos: 777 - 022 = 755 (rwxr-xr-x)

# Cambiar umask (solo para la sesión actual)
umask 0077   # Archivos: 600, Directorios: 700 (solo el dueño)
```

---

## La Regla de Oro: NUNCA USAR 777

`chmod 777` significa que **cualquiera** puede leer, escribir y ejecutar tu archivo.

- **Mala práctica:** `chmod 777 script.sh`
- **Buena práctica:** Usar grupos y permisos mínimos necesarios.
