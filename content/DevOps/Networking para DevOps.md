[[0. Índice DevOps]]

# Networking para DevOps

## 1. DNS — Más Allá de lo Básico

### Tipos de Registros

| Tipo      | Función                            | Ejemplo                                |
| --------- | ---------------------------------- | -------------------------------------- |
| **A**     | Dominio → IPv4                     | `app.com → 93.184.216.34`              |
| **AAAA**  | Dominio → IPv6                     | `app.com → 2606:2800:220:1:248:...`    |
| **CNAME** | Alias de otro dominio              | `www.app.com → app.com`                |
| **MX**    | Servidor de email                  | `app.com → mail.app.com`               |
| **TXT**   | Texto libre (SPF, verificación)    | `app.com → "v=spf1 include:..."`       |
| **NS**    | Nameservers del dominio            | `app.com → ns1.provider.com`           |
| **SRV**   | Servicio con puerto                | `_sip._tcp.app.com → 5060 sip.app.com` |
| **CAA**   | Qué CAs pueden emitir certificados | `app.com → 0 issue "letsencrypt.org"`  |

### Registros para DevOps comunes

```bash
# A record para tu servidor
app.com         A       93.184.216.34

# CNAME para subdominio
www.app.com     CNAME   app.com
api.app.com     CNAME   lb.app.com

# Wildcard (todos los subdominios)
*.app.com       A       93.184.216.34

# TXT para verificación (Google, Let's Encrypt)
app.com         TXT     "google-site-verification=abc123"
_acme-challenge.app.com  TXT  "valor-del-challenge"

# MX para email
app.com         MX  10  mail.app.com
app.com         MX  20  mail2.app.com    # Backup (prioridad mayor = menos preferido)

# SPF, DKIM, DMARC (para que tus emails no vayan a spam)
app.com         TXT     "v=spf1 include:_spf.google.com ~all"
```

### DNS en AWS (Route 53)

```bash
# Crear hosted zone
aws route53 create-hosted-zone --name app.com --caller-reference $(date +%s)

# Listar records
aws route53 list-resource-record-sets --hosted-zone-id Z12345

# Route 53 features especiales:
# - Weighted routing (50% a server A, 50% a server B)
# - Latency routing (responde con el server más cercano)
# - Failover routing (health check + failover automático)
# - Geolocation routing (por país/continente)
```

---

## 2. VPN — Acceso Seguro

### WireGuard (moderno, rápido)

```bash
# Instalar
sudo apt install wireguard

# Generar claves (servidor)
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Generar claves (cliente)
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

```ini
# /etc/wireguard/wg0.conf (SERVIDOR)
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>

# Habilitar forwarding
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

```ini
# /etc/wireguard/wg0.conf (CLIENTE)
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private_key>
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = IP-SERVIDOR:51820
AllowedIPs = 10.0.0.0/24          # Solo tráfico a la VPN
# AllowedIPs = 0.0.0.0/0          # Todo el tráfico por la VPN
PersistentKeepalive = 25
```

```bash
# Iniciar
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# Ver estado
sudo wg show

# Parar
sudo wg-quick down wg0
```

---

## 3. Service Mesh (Kubernetes Avanzado)

Un service mesh maneja la comunicación entre microservicios automáticamente.

```
Sin service mesh:
  App A ---(HTTP directo)--→ App B
  Problemas: retry, timeout, mTLS, observabilidad... todo manual

Con service mesh (Istio/Linkerd):
  App A → Sidecar Proxy → Sidecar Proxy → App B
  El proxy maneja: mTLS, retry, circuit breaker, observabilidad
```

### Istio (el más popular)

```bash
# Instalar
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

istioctl install --set profile=demo
kubectl label namespace default istio-injection=enabled
```

### Linkerd (más simple)

```bash
# Instalar
curl --proto '=https' -sL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
linkerd check

# Inyectar en un namespace
kubectl annotate namespace mi-app linkerd.io/inject=enabled
```

### Cuándo usar Service Mesh

```
NO lo necesitás si:
  - Tenés pocos servicios (< 5)
  - No necesitás mTLS entre servicios
  - Podés manejar retry/timeout en la app

SÍ lo necesitás si:
  - Muchos microservicios (> 10)
  - Necesitás mTLS obligatorio (compliance, zero trust)
  - Querés observabilidad sin cambiar código
  - Necesitás traffic splitting (canary deployments)
```

---

## 4. Modelos de Red en Kubernetes

```
# ClusterIP: red interna del cluster
Pod → Service (ClusterIP) → Pods destino
10.96.0.0/12 (por defecto)

# DNS interno
mi-servicio.mi-namespace.svc.cluster.local

# NodePort: expone en cada nodo
Host:30080 → Service → Pods

# LoadBalancer: IP externa (cloud)
IP-Externa:80 → Service → Pods

# Ingress: HTTP routing
app.com → Ingress Controller → Service → Pods
api.com → Ingress Controller → Service → Pods
```

### Network Policies (firewall entre pods)

```yaml
# Solo permitir tráfico del frontend al backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
```

---

## 5. Proxy Reverso y Forward Proxy

```
Forward Proxy:
  Cliente → Proxy → Internet
  (el proxy sale a internet en nombre del cliente)
  Uso: cache, filtrado, privacidad

Reverse Proxy:
  Internet → Proxy → Servidor interno
  (el proxy recibe requests y los distribuye)
  Uso: load balancing, SSL termination, cache
  Ejemplos: nginx, HAProxy, Traefik, Envoy
```

### Traefik (Alternativa a nginx para K8s)

```yaml
# Traefik como Ingress en K8s (auto-descubre servicios)
# Con Helm:
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace
```

---

## 6. Troubleshooting de Red

```bash
# ¿DNS resuelve?
dig app.ejemplo.com
nslookup app.ejemplo.com
host app.ejemplo.com

# ¿El puerto está abierto?
nc -zv host 80
telnet host 80

# ¿Hay conectividad?
ping host
traceroute host
mtr host

# ¿Qué está escuchando?
ss -tlnp

# ¿Qué pasa en la red? (captura de paquetes)
sudo tcpdump -i eth0 port 80 -A

# ¿Qué rutas tiene?
ip route show

# ¿El certificado está bien?
openssl s_client -connect host:443

# ¿DNS del cluster K8s funciona?
kubectl exec -it pod -- nslookup kubernetes.default

# ¿Hay conectividad entre pods?
kubectl exec -it pod-a -- curl http://servicio-b:80

# ¿El servicio está resolviendo?
kubectl exec -it pod -- nslookup mi-servicio.mi-namespace.svc.cluster.local
```

---

## 7. Conceptos Clave para Entrevistas

```
CIDR: 10.0.0.0/16 = 65536 IPs (10.0.0.0 - 10.0.255.255)
      10.0.1.0/24 = 256 IPs  (10.0.1.0 - 10.0.1.255)
      10.0.1.0/28 = 16 IPs   (10.0.1.0 - 10.0.1.15)

Subnetting: Dividir una red grande en redes más chicas
NAT: Traducir IPs privadas a públicas (para salir a internet)
VLAN: Separar tráfico en una misma red física
mTLS: TLS mutuo (ambos lados verifican certificados)
Zero Trust: No confiar en nada, verificar todo (aunque sea tráfico interno)
CDN: Content Delivery Network (cache en edge locations cerca del usuario)
Anycast: Misma IP en múltiples ubicaciones (DNS, CDN)
BGP: Protocolo de routing entre redes grandes (providers)
```
