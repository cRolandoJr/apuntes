[[0. General Tips]]

# SSH — Acceso Remoto Seguro

Mantener la seguridad de los servidores es una de las tareas principales del SysAdmin. Aquí están los pasos para proteger el acceso SSH.

## 1. Flujo de Configuración Inicial

**Paso 0:** Crear usuarios en el servidor ([[Administración de cuentas y usuarios]]) y agregarlos a los grupos necesarios (**sudo** en Debian, **wheel** en Arch). Crear una cuenta de administrador para no entrar directamente como root.

**Paso 1:** Generar un par de llaves SSH (si aún no tenés):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub Usuario@IPdelServidor
```

> El par de llaves ya debe haber sido creado. Si no existía:

```bash
ssh-keygen -t ed25519 -C "tu@correo.com"
```

**Paso 2:** Conectarse una vez más y verificar que no pide contraseña. Luego editar `/etc/ssh/sshd_config` para deshabilitar acceso por contraseña y el login directo como root.

Buscar y descomentar (quitar el `#`) estas líneas:

- `PermitRootLogin no`
- `PasswordAuthentication no`

**Paso 3:** Recargar el demonio SSH:

```bash
systemctl reload sshd
```

> **VITAL:** No cerrar la sesión actual. Abrir otra terminal y probar el acceso antes de desconectar. Si la copia de la llave falló, podrías quedar afuera del servidor.

**Nota sobre actualizaciones:**
![[Pasted image 20260227161921.png]]
Si esto aparece al actualizar paquetes: **no instales la nueva versión del config**. Mantené la versión actual (que tiene tu configuración SSH). Si no recordás qué cambiaste, revisá las diferencias entre versiones antes de decidir.

## 2. Cambiar Puerto SSH (por defecto: 22)

### Paso A: Editar la configuración

```bash
sudo vim /etc/ssh/sshd_config
```

### Paso B: Asignar el nuevo puerto

Buscar `#Port 22`, descomentar y cambiar a un puerto alto (entre 1024 y 65535):

```
Port 45678
```

### Paso C: Configurar el Firewall (ANTES de reiniciar SSH)

**Antes** de reiniciar SSH, abrir el nuevo puerto en el firewall. Si no, quedás afuera:

Si usás **UFW**:

```bash
sudo ufw allow 45678/tcp
sudo ufw delete allow 22/tcp
```

### Paso D: Recargar SSH

```bash
sudo systemctl reload sshd
```

### Paso E: Conectarse con el nuevo puerto

> **Recordar:** Probar en una terminal nueva antes de cerrar la sesión actual.

```bash
ssh -p 45678 tu_usuario@IP_del_servidor
```

---

## 3. Configuración de Firewall para SSH con Otros Firewalls

### Firewalld (CentOS/RHEL/Fedora)

**Abrir el nuevo puerto:**

```bash
sudo firewall-cmd --permanent --add-port=45678/tcp
```

**Cerrar el viejo:** Con Firewalld, el puerto 22 suele estar habilitado por defecto bajo el nombre del servicio `ssh`. Así que lo removemos directamente:

```bash
sudo firewall-cmd --permanent --remove-service=ssh
```

**Aplicar los cambios:**

```bash
sudo firewall-cmd --reload
```

### iptables (La vieja escuela)

**Abrir el nuevo puerto:** Le decimos a la cadena de entrada (INPUT) que acepte (ACCEPT) el protocolo tcp en el puerto 45678.

```bash
sudo iptables -A INPUT -p tcp --dport 45678 -j ACCEPT
```

**Cerrar el viejo:** Aquí tienes dos opciones. Si tenías una regla que aceptaba el puerto 22, la borras (cambiando la `-A` de Add por `-D` de Delete):

```bash
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT
```

_(Nota: A diferencia de UFW o Firewalld, iptables olvida estas reglas si reiniciás el servidor. Usá `iptables-persistent` para guardarlas)._

### nftables (El sucesor de iptables)

**Abrir el nuevo puerto:**

Le indicamos a la cadena de entrada que acepte el tráfico TCP dirigido al puerto 45678:

```bash
sudo nft add rule inet filter input tcp dport 45678 accept
```

**Cerrar el puerto viejo (método del Handle):**

En nftables, la forma más segura de borrar una regla es encontrar su **handle** (identificador único):

Listar reglas con handles:

```bash
sudo nft -a list ruleset
```

Buscar la regla del puerto 22 y anotar el número de handle (ej: `handle 5`):

```bash
sudo nft delete rule inet filter input handle 5
```

**Guardar los cambios** (sin esto, se pierden al reiniciar):

```bash
sudo nft list ruleset > /etc/nftables.conf
```

> La ruta `/etc/nftables.conf` es la estándar. Asegurate de que el servicio `nftables` esté habilitado: `sudo systemctl enable nftables`

---

**Relacionado:** [[fail2ban]] | [[Redes & Troubleshooting]]
