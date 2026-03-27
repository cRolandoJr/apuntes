[[0. General Tips]]

## 1. Archivos Clave del Sistema

| Archivo       | Contenido                                            |
| ------------- | ---------------------------------------------------- |
| `/etc/passwd` | Usuarios: `username:x:uid:gid:comentario:home:shell` |
| `/etc/shadow` | Contraseñas encriptadas de los usuarios              |
| `/etc/group`  | Grupos del sistema                                   |

---

## 2. Crear Usuarios

```bash
# Crear usuario con home y shell bash
sudo useradd -m -s /bin/bash nombre_usuario

# Asignar contraseña
sudo passwd nombre_usuario
```

**Opciones de `useradd`:**

| Opción             | Función                            |
| ------------------ | ---------------------------------- |
| `-m`               | Crear directorio home              |
| `-d /ruta`         | Especificar otro directorio home   |
| `-c "comentario"`  | Descripción del usuario            |
| `-s /bin/bash`     | Shell de login                     |
| `-G grupo1,grupo2` | Grupos secundarios (deben existir) |
| `-g grupo`         | Grupo primario (debe existir)      |

**Ejemplo completo:**

```bash
useradd -m -d /home/john -c "C++ Developer" -s /bin/bash -G sudo,adm,mail john
```

---

## 3. Dar Permisos de Administrador (sudo)

```bash
# Debian/Ubuntu → grupo "sudo"
sudo usermod -aG sudo nombre_usuario

# Arch Linux / CentOS / RHEL → grupo "wheel"
sudo usermod -aG wheel nombre_usuario
```

---

## 4. Modificar Usuarios

```bash
# Agregar a grupos secundarios (la -a es vital: sin ella REEMPLAZA los grupos)
sudo usermod -aG grupo1,grupo2 nombre_usuario
```

> **Cuidado:** `usermod -G grupo usuario` (sin `-a`) **quita** al usuario de todos los demás grupos secundarios. Siempre usar `-aG`.

---

## 5. Eliminar Usuarios y Grupos

```bash
# Eliminar usuario y su home
sudo userdel -r nombre_usuario

# Crear grupo
sudo groupadd nombre_grupo

# Eliminar grupo
sudo groupdel nombre_grupo
```

---

## 6. Consultar Información

```bash
# Ver todos los grupos del sistema
cat /etc/group

# Ver grupos del usuario actual
groups

# Ver grupos de un usuario específico
groups nombre_usuario

# Ver UID, GID y grupos
id
id nombre_usuario
```

---

## 7. Monitorear Usuarios Conectados

```bash
# Quién está conectado (con headers)
who -H

# Quién está conectado + qué proceso ejecutan + carga del sistema
w

# Tiempo de actividad del servidor
uptime

# Usuario actual
whoami

# Historial de sesiones (quién entró y cuándo)
last

# Historial de un usuario específico
last -u nombre_usuario
```

---
