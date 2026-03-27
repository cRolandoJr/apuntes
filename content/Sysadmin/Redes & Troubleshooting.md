[[0. General Tips]]

# Redes y Diagnóstico

## 1. Metodología de Troubleshooting de Red

Seguir este orden cuando algo no conecta:

```
1. ¿Tengo IP?                    → ip a
2. ¿Tengo salida a internet?      → ping 8.8.8.8
3. ¿Funciona el DNS?              → ping google.com
4. ¿El servicio escucha?          → ss -tulpn
5. ¿El firewall bloquea?          → ufw status
6. ¿El puerto remoto está abierto? → nc -zv IP PUERTO
```

---

## 2. Comandos de Diagnóstico

```bash
# Ver interfaces de red y sus IPs (NO usar ifconfig, es obsoleto)
ip a

# Ver solo las IPs asignadas
ip -br a

# Ver tabla de rutas
ip route

# Ver gateway por defecto
ip route | grep default

# Probar conectividad básica
ping 8.8.8.8          # Prueba red (sin DNS)
ping google.com       # Prueba red + DNS
ping -c 4 8.8.8.8     # Solo 4 paquetes

# Trazar ruta (ver por dónde pasa el tráfico)
traceroute google.com
# O la versión rápida:
mtr google.com

# Consultar DNS
dig google.com            # Resolución DNS detallada
dig google.com +short     # Solo la IP
nslookup google.com       # Alternativa más simple

# Si DNS falla, verificar/editar resolvers
cat /etc/resolv.conf
# Agregar: nameserver 8.8.8.8
```

---

## 3. Puertos y Conexiones

### El mnemónico: **TULPN** (`ss -tulpn`)

| Letra | Significado                       |
| ----- | --------------------------------- |
| **T** | TCP                               |
| **U** | UDP                               |
| **L** | Listening (escuchando)            |
| **P** | Process (qué programa lo usa)     |
| **N** | Numeric (muestra IPs, no nombres) |

```bash
# Ver qué puertos están abiertos y quién escucha
sudo ss -tulpn

# Filtrar por un puerto específico
sudo ss -tulpn | grep :80

# Ver todas las conexiones activas (no solo listening)
ss -tan
```

### Verificar puertos remotos

```bash
# ¿El servidor remoto tiene el puerto abierto?
nc -zv IP_DESTINO 80

# Con curl
curl -v http://IP_DESTINO

# Con timeout (no quedarse esperando)
nc -zv -w 3 IP_DESTINO 443
```

---

## 4. Firewall (UFW — Debian/Ubuntu)

```bash
# Ver estado del firewall
sudo ufw status verbose

# IMPORTANTE: Permitir SSH PRIMERO antes de activar
sudo ufw allow 22/tcp

# Activar firewall
sudo ufw enable

# Permitir puertos comunes
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 8080/tcp    # App custom

# Permitir desde una IP específica
sudo ufw allow from 192.168.1.100

# Bloquear una IP
sudo ufw deny from 203.0.113.50

# Eliminar una regla
sudo ufw delete allow 80/tcp

# Recargar reglas
sudo ufw reload

# Desactivar firewall
sudo ufw disable
```

> **Regla de supervivencia:** SIEMPRE permitir SSH antes de activar el firewall. Si no, quedás afuera del servidor.

---

## 5. Captura de Tráfico (tcpdump)

```bash
# Capturar todo el tráfico en una interfaz
sudo tcpdump -i eth0

# Solo tráfico en un puerto específico
sudo tcpdump -i eth0 port 80

# Solo tráfico desde/hacia una IP
sudo tcpdump -i eth0 host 192.168.1.50

# Guardar captura a un archivo (para analizar con Wireshark)
sudo tcpdump -i eth0 -w captura.pcap
```

---

**Relacionado:** [[nginx]] | [[SSH]] | [[fail2ban]]
