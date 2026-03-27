[[0. General Tips]]

# Gestión de Paquetes

## 1. APT (Debian / Ubuntu)

### Operaciones Básicas

```bash
# Actualizar lista de paquetes disponibles
sudo apt update

# Actualizar todos los paquetes instalados
sudo apt upgrade

# Actualización completa (puede agregar/quitar paquetes si es necesario)
sudo apt full-upgrade

# Instalar un paquete
sudo apt install nombre_paquete

# Instalar sin preguntar confirmación
sudo apt install -y nombre_paquete

# Desinstalar (mantiene archivos de configuración)
sudo apt remove nombre_paquete

# Desinstalar + borrar configuración
sudo apt purge nombre_paquete

# Limpiar paquetes huérfanos (dependencias que ya no se necesitan)
sudo apt autoremove
```

### Búsqueda e Información

```bash
# Buscar un paquete
apt search nombre

# Ver información de un paquete
apt show nombre_paquete

# Ver qué paquetes están instalados
apt list --installed

# Ver qué paquetes se pueden actualizar
apt list --upgradable

# Ver a qué paquete pertenece un archivo
dpkg -S /ruta/al/archivo

# Ver qué archivos instaló un paquete
dpkg -L nombre_paquete
```

### Repositorios

```bash
# Los repos están en:
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# Agregar un repositorio PPA (Ubuntu)
sudo add-apt-repository ppa:nombre/ppa
sudo apt update

# Agregar repo manualmente
echo "deb http://repo.ejemplo.com/ubuntu focal main" | sudo tee /etc/apt/sources.list.d/ejemplo.list

# Agregar clave GPG de un repo (necesario para repos externos)
curl -fsSL https://repo.ejemplo.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/ejemplo.gpg
```

### Pinning (Fijar versiones)

Para evitar que un paquete se actualice automáticamente:

```bash
# Bloquear actualización de un paquete
sudo apt-mark hold nombre_paquete

# Desbloquear
sudo apt-mark unhold nombre_paquete

# Ver paquetes bloqueados
apt-mark showhold
```

### Caché y Limpieza

```bash
# Limpiar caché de paquetes descargados (libera espacio)
sudo apt clean

# Limpiar solo paquetes que ya no se pueden descargar
sudo apt autoclean

# Ver cuánto espacio ocupa la caché
du -sh /var/cache/apt/archives/
```

---

## 2. Pacman (Arch Linux)

```bash
# Sincronizar repos y actualizar todo
sudo pacman -Syu

# Instalar paquete
sudo pacman -S nombre_paquete

# Desinstalar paquete + dependencias huérfanas
sudo pacman -Rns nombre_paquete

# Buscar paquete
pacman -Ss nombre

# Info de un paquete
pacman -Si nombre_paquete

# Ver paquetes instalados
pacman -Q

# Ver paquetes huérfanos
pacman -Qdt

# Limpiar huérfanos
sudo pacman -Rns $(pacman -Qdtq)

# Limpiar caché (mantener solo la última versión)
sudo pacman -Sc

# Listar archivos de un paquete instalado
pacman -Ql nombre_paquete

# ¿A qué paquete pertenece este archivo?
pacman -Qo /ruta/al/archivo
```

---

## 3. DNF (Fedora / RHEL / Rocky / AlmaLinux)

```bash
# Actualizar todo
sudo dnf upgrade

# Instalar
sudo dnf install nombre_paquete

# Desinstalar
sudo dnf remove nombre_paquete

# Buscar
dnf search nombre

# Info
dnf info nombre_paquete

# Ver qué paquete provee un comando
dnf provides */comando

# Listar repos habilitados
dnf repolist

# Agregar repo
sudo dnf config-manager --add-repo URL

# Limpiar caché
sudo dnf clean all
```

---

## 4. Instalar desde Código Fuente

Cuando un paquete no está en los repos (último recurso):

```bash
# Patrón clásico
tar -xvf programa-1.0.tar.gz
cd programa-1.0
./configure
make
sudo make install

# Para desinstalar (si el Makefile lo soporta)
sudo make uninstall
```

> **Advertencia:** Los paquetes instalados desde fuente no se gestionan con el package manager. Preferí siempre repos oficiales o paquetes .deb/.rpm.

---

## 5. Buenas Prácticas

- **Siempre `apt update` antes de `apt install`** — Si no, podés instalar versiones viejas
- **No mezclar repos de distintas versiones** — Puede romper dependencias
- **Los servidores de producción no usan PPAs** — Solo repos oficiales y repos del proveedor
- **Documentar qué repos externos agregaste** — Para poder reproducir el setup
- **Antes de actualizar un servidor de producción**, probar en staging primero
