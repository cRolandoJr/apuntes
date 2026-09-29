[[Redes & Troubleshooting]]

# UFW — Firewall en Debian/Ubuntu

UFW (*Uncomplicated Firewall*) no es un firewall: es un **frontend**. El firewall real
es netfilter, en el kernel. ufw traduce tus comandos a reglas y las persiste en
archivos para que sobrevivan al reboot.

En Ubuntu 24.04 esa traducción pasa por `iptables-nft`, no por el iptables clásico:

```bash
iptables --version
# iptables v1.8.10 (nf_tables)   ← "iptables" acá es un shim sobre nftables
```

Importa para no confundirse leyendo tutoriales viejos: las reglas que crea ufw se ven
con `iptables -L`, pero por debajo viven en nftables.

> Verificado sobre ufw 0.36.2 / Ubuntu Server 24.04.4 LTS.

---

## 1. Lo que más confunde: la CLI **es** la fuente de verdad

Al revés que SSH, donde `sshd_config` manda y no hay CLI que lo escriba, en ufw los
archivos se dividen en dos grupos que se tratan distinto:

**Los que editás vos:**

| Archivo | Qué define |
| --- | --- |
| `/etc/default/ufw` | políticas por defecto: `DEFAULT_INPUT_POLICY`, `..._OUTPUT_...`, `..._FORWARD_...`, `IPV6` |
| `/etc/ufw/ufw.conf` | `ENABLED` (arranca al boot) y `LOGLEVEL` |
| `/etc/ufw/before.rules` · `after.rules` | iptables crudo, para lo que la CLI no expresa (se aplican antes/después de tus reglas) |
| `/etc/ufw/applications.d/*` | perfiles con nombre |
| `/etc/ufw/sysctl.conf` | parámetros de red del kernel |

**Los que genera ufw — nunca a mano:** `/etc/ufw/user.rules` y `user6.rules`. Ahí
aterriza cada `ufw allow`/`limit`. Son texto legible, pero el próximo comando `ufw`
los reescribe entero.

Y ninguno de los dos grupos es el estado real: el estado real está en el kernel. Los
archivos son solo cómo sobrevive a un reboot.

---

## 2. El orden que no se negocia

```bash
sudo ufw limit 45678/tcp comment 'ssh'   # 1. la regla PRIMERO
sudo ufw enable                          # 2. el enable DESPUÉS
```

`ufw enable` aplica `deny incoming` **en el acto**. Si lo habilitás antes de permitir
tu puerto SSH, la sesión se corta en ese mismo comando y no volvés a entrar por red.

Si trabajás sobre una VM local, la red de escape es la consola serie, que no depende
del firewall:

```bash
virsh -c qemu:///system console <vm>     # salir: Ctrl+]
```

En un VPS el equivalente es la consola web del proveedor. Si no tenés ninguna de las
dos, no habilites ufw sin la regla puesta.

---

## 3. `limit` en vez de `allow` para SSH

```bash
sudo ufw limit 45678/tcp comment 'ssh'
```

`limit` agrega rate-limiting nativo: **deniega si una IP intenta 6 o más conexiones en
30 segundos** (`man ufw`). Cubre buena parte de lo que se le pediría a
[[fail2ban]] para SSH sin sumar un daemon.

No lo reemplaza del todo: fail2ban banea por *fallos de autenticación* leídos del log
(más preciso, y con historial), mientras que `limit` cuenta *conexiones* sin importar
si autenticaron bien. Para un server chico, `limit` solo ya es una mejora enorme sobre
`allow`.

**El `comment` no es decorativo.** Aparece en `ufw status` y es la diferencia entre
entender tu propio firewall en seis meses o no.

---

## 4. Nunca `ufw allow ssh`

Ese perfil es literalmente el puerto 22:

```bash
cat /etc/ufw/applications.d/openssh-server
# [OpenSSH]
# ports=22/tcp
```

Si moviste SSH de puerto, `ufw allow ssh` te abre una puerta que no usás y deja
cerrada la real. Siempre el número explícito.

```bash
sudo ufw app list             # perfiles disponibles
sudo ufw app info OpenSSH     # qué puertos abre uno en concreto
```

---

## 5. Ver y borrar reglas

```bash
sudo ufw status verbose       # estado + políticas por defecto + logging
sudo ufw status numbered      # con índice, que es lo que necesitás para borrar
sudo ufw delete 2             # borrar por número (el índice se recalcula: de a una)
sudo ufw delete limit 45678/tcp   # o repitiendo la regla exacta
```

Borrar por número es lo práctico, pero **los índices se renumeran después de cada
borrado**. Si tenés que sacar varias, volvé a correr `status numbered` entre una y
otra, o borrá de mayor a menor.

Para insertar en una posición concreta (el orden importa: gana la primera que matchea):

```bash
sudo ufw insert 1 deny from 203.0.113.50
sudo ufw prepend deny from 203.0.113.50    # equivalente a insert 1
```

---

## 6. IPv6: el agujero fácil de dejar

`/etc/default/ufw` trae `IPV6=yes`, así que cada regla se aplica en las dos familias.
Cuando lo hace, la CLI lo dice:

```
Rules updated
Rules updated (v6)      ← si esta línea no aparece, tenés medio firewall
```

Y en el estado se ven las dos:

```
45678/tcp        LIMIT IN    Anywhere
45678/tcp (v6)   LIMIT IN    Anywhere (v6)
```

Vale mirarlo. Es fácil terminar con un host cerrado por v4 y abierto por v6.

---

## 7. Logging

```bash
sudo ufw logging low          # off | low | medium | high | full
```

Los niveles no son "más o menos verborrágico" nomás:

- **low** — solo paquetes bloqueados que no matchean la política, más las reglas logueadas
- **medium** — low + permitidos que no matchean la política + INVALID + conexiones nuevas
- **high** — medium sin rate-limiting, más todos los paquetes con rate-limiting
- **full** — todo, sin rate-limiting

`medium` para arriba llena el disco rápido en un server con tráfico. `low` es el
default y el correcto salvo que estés diagnosticando algo puntual.

Los logs van a `/var/log/ufw.log` vía rsyslog (Ubuntu Server lo trae; las imágenes
cloud/minimal no, y ahí hay que leer el journal).

---

## 8. Gotchas verificados

**Con ufw inactivo, `status verbose` no muestra las reglas.** Imprime solo
`Status: inactive`, aunque `ufw limit ...` ya haya contestado `Rules updated`. Las
reglas están escritas en `/etc/ufw/user.rules`, pero es `640 root:root`: con el
firewall apagado, la única forma de auditarlas es leer el archivo con sudo. Invita a
creer que el comando no tuvo efecto y correrlo de nuevo.

**`systemctl is-enabled ufw` no dice si el firewall está filtrando.** Devuelve
`enabled` en una Ubuntu recién instalada, donde ufw está *inactivo*. Eso es la unit de
systemd, no la política. El estado real:

```bash
sudo ufw status              # ← esto
grep ENABLED /etc/ufw/ufw.conf
```

**`ufw enable` hace dos cosas.** Levanta el firewall *y* escribe `ENABLED=yes` en
`ufw.conf`, o sea que también lo activa al boot. No hace falta un `systemctl enable`
aparte.

**`ufw reset` borra todo sin preguntar dos veces.** Deja backups de las reglas en
`/etc/ufw/*.rules.<timestamp>`, que es lo único que te salva.

---

## 9. Verificar desde afuera, no desde adentro

Un `ufw status` que dice `active` prueba que ufw cree estar activo. Lo que importa es
si el camino real funciona, y eso solo se ve desde otro host:

```bash
# el puerto que debe estar abierto: responde
ssh -p 45678 usuario@192.168.122.35 'echo ok'

# el viejo, que debe estar cerrado
timeout 6 bash -c 'cat </dev/null >/dev/tcp/192.168.122.35/22'

# un puerto donde NO escucha nadie — el control del experimento
timeout 6 bash -c 'cat </dev/null >/dev/tcp/192.168.122.35/80'
```

El tercero es el que hace válida la prueba: si el 80 también rebota, sabés que el
`deny incoming` está actuando y que el 22 cerrado no es simplemente "no hay servicio
escuchando ahí".

---

## 10. Reglas de uso frecuente

```bash
# por puerto
sudo ufw allow 80/tcp comment 'http'
sudo ufw allow 443/tcp comment 'https'

# solo desde una red (lo más útil en un lab o una LAN)
sudo ufw allow from 192.168.122.0/24 to any port 5432 proto tcp comment 'postgres LAN'

# bloquear un origen
sudo ufw deny from 203.0.113.50

# deny vs reject: deny descarta en silencio (el cliente espera y timeoutea),
# reject contesta "cerrado" (falla al toque). deny es el default y expone menos.
sudo ufw reject from 203.0.113.50

# tráfico ruteado (si la máquina hace de router/NAT; NO aplica al tráfico propio)
sudo ufw route allow in on eth0 out on eth1 to any port 443
```

**El tráfico saliente no necesita reglas** mientras `DEFAULT_OUTPUT_POLICY="ACCEPT"`.
Un agente que reporta a un servidor central (Wazuh, Prometheus, backups) es saliente:
no hay que abrir nada de entrada para que funcione.

---

## Ver también

- [[SSH]] — cambiar el puerto y el orden correcto respecto del firewall
- [[fail2ban]] — baneo por fallos de autenticación, complementario a `limit`
- [[Hardening y Seguridad Avanzada]] — dónde entra el firewall en el checklist
- [[Redes & Troubleshooting]] — diagnóstico de puertos y conectividad
