[[0. General Tips]]

# Configuración de Red Avanzada

## 1. Interfaces de Red

```bash
# Ver todas las interfaces y sus IPs
ip a

# Versión resumida
ip -br a

# Ver solo interfaces activas (UP)
ip -br link show up

# Ver tabla de rutas
ip route

# Ver gateway por defecto
ip route | grep default

# Ver vecinos (tabla ARP)
ip neigh
```

---

## 2. Configurar IP Estática

### Netplan (Ubuntu 18.04+)

Archivo: `/etc/netplan/01-netcfg.yaml` (o similar)

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18: # Nombre de tu interfaz
      addresses:
        - 192.168.1.50/24 # IP estática / máscara
      routes:
        - to: default
          via: 192.168.1.1 # Gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
# Aplicar cambios
sudo netplan apply

# Validar antes de aplicar (rollback automático si perdés conexión)
sudo netplan try
```

### nmcli (NetworkManager — Fedora, RHEL, Desktop)

```bash
# Ver conexiones
nmcli con show

# Configurar IP estática
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.method manual

# Activar
sudo nmcli con up "Wired connection 1"

# Volver a DHCP
sudo nmcli con mod "Wired connection 1" ipv4.method auto
sudo nmcli con up "Wired connection 1"
```

### Archivo clásico (Debian sin Netplan)

Archivo: `/etc/network/interfaces`

```
auto ens18
iface ens18 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

```bash
sudo systemctl restart networking
```

---

## 3. DNS

### Resolución

```bash
# Ver qué DNS estás usando
cat /etc/resolv.conf

# En sistemas con systemd-resolved
resolvectl status

# Cambiar DNS temporalmente
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# Cambiar DNS permanentemente → Hacerlo en netplan o nmcli (no en resolv.conf directo)
```

### Archivo `/etc/hosts` (Resolución local)

```bash
sudo vim /etc/hosts
```

```
127.0.0.1       localhost
192.168.1.10    servidor-web    web
192.168.1.20    servidor-db     db
192.168.1.30    servidor-app    app
```

> Esto permite usar `ssh web` en vez de `ssh 192.168.1.10`. Se resuelve antes que el DNS.

### Hostname

```bash
# Ver hostname actual
hostnamectl

# Cambiar hostname
sudo hostnamectl set-hostname mi-servidor

# También actualizar /etc/hosts
sudo vim /etc/hosts
# Cambiar la línea que tiene el hostname viejo
```

---

## 4. Manipulación de Interfaces

```bash
# Activar/desactivar una interfaz
sudo ip link set ens18 up
sudo ip link set ens18 down

# Agregar una IP temporal a una interfaz
sudo ip addr add 10.0.0.5/24 dev ens18

# Quitar una IP
sudo ip addr del 10.0.0.5/24 dev ens18

# Agregar ruta estática temporal
sudo ip route add 10.10.0.0/24 via 192.168.1.1

# Agregar ruta estática permanente (Netplan)
# En el archivo netplan:
#   routes:
#     - to: 10.10.0.0/24
#       via: 192.168.1.1
```

---

## 5. VLANs

VLANs permiten segmentar tráfico en la misma interfaz física.

### Con Netplan

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
  vlans:
    vlan100:
      id: 100
      link: ens18
      addresses:
        - 10.100.0.5/24
    vlan200:
      id: 200
      link: ens18
      addresses:
        - 10.200.0.5/24
```

### Manual

```bash
# Crear VLAN
sudo ip link add link ens18 name ens18.100 type vlan id 100
sudo ip addr add 10.100.0.5/24 dev ens18.100
sudo ip link set ens18.100 up
```

---

## 6. Network Bonding / Teaming

Combinar múltiples interfaces para redundancia o ancho de banda.

### Con Netplan (modo active-backup)

```yaml
network:
  version: 2
  bonds:
    bond0:
      interfaces:
        - ens18
        - ens19
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      parameters:
        mode: active-backup # Si ens18 cae, ens19 toma el control
        primary: ens18
```

| Modo            | Uso                                                  |
| --------------- | ---------------------------------------------------- |
| `active-backup` | Redundancia — solo una interfaz activa               |
| `balance-rr`    | Round-robin — usa ambas (requiere switch compatible) |
| `802.3ad`       | LACP — agregación (requiere switch compatible)       |

---

## 7. Diagnóstico Avanzado

```bash
# Ver estadísticas de una interfaz
ip -s link show ens18

# Ver errores y drops
ethtool -S ens18

# Ver velocidad y duplex negociados
ethtool ens18

# Ver la tabla de conexiones del kernel (NAT, conntrack)
sudo conntrack -L

# Ver sockets en estado TIME_WAIT (muchos = posible leak)
ss -tan | grep TIME-WAIT | wc -l

# Test de ancho de banda entre dos servidores
# En el servidor: iperf3 -s
# En el cliente:  iperf3 -c IP_SERVIDOR
```
