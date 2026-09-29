[[0. Índice DevOps]]

# Ansible

## 1. ¿Qué es Ansible?

Herramienta de **automatización de configuración**. Vos describís el estado deseado de tus servidores y Ansible se encarga de que lleguen a ese estado.

- **Agentless:** No necesita instalar nada en los servidores (usa SSH)
- **Idempotente:** Podés ejecutarlo 100 veces y solo hace cambios si son necesarios
- **Declarativo:** Describís QUÉ querés, no CÓMO

```
Antes de Ansible:
  ssh server1 "apt update && apt install nginx"
  ssh server2 "apt update && apt install nginx"
  ssh server3 "apt update && apt install nginx"

Con Ansible:
  ansible all -m apt -a "name=nginx state=present"
  (se ejecuta en los 3 servers al mismo tiempo)
```

---

## 2. Instalación

```bash
# En tu máquina de control (no en los servidores)
sudo apt install ansible

# O con pip
pip install ansible

# Verificar
ansible --version
```

---

## 3. Inventario (¿Dónde están mis servidores?)

```ini
# /etc/ansible/hosts  o  ./inventory/hosts.ini

# Formato INI
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[dbservers]
db1 ansible_host=192.168.1.20

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_port=22
```

```yaml
# Formato YAML (inventory/hosts.yml) — más moderno
all:
  vars:
    ansible_user: deploy
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
        web2:
          ansible_host: 192.168.1.11
    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.1.20
```

```bash
# Verificar inventario
ansible-inventory -i inventory/hosts.yml --list

# Probar conexión a todos
ansible all -i inventory/hosts.yml -m ping
```

---

## 4. Comandos Ad-Hoc (Rápidos)

```bash
# Ping a todos los servidores
ansible all -m ping

# Ejecutar comando
ansible webservers -m shell -a "uptime"
ansible all -m shell -a "df -h"

# Instalar paquete
ansible webservers -m apt -a "name=nginx state=present" --become

# Copiar archivo
ansible all -m copy -a "src=./config.conf dest=/etc/app/config.conf" --become

# Reiniciar servicio
ansible webservers -m service -a "name=nginx state=restarted" --become

# Ver facts (información del servidor)
ansible web1 -m setup
ansible web1 -m setup -a "filter=ansible_os_family"
```

| Flag                | Significado               |
| ------------------- | ------------------------- |
| `-m`                | Módulo a usar             |
| `-a`                | Argumentos del módulo     |
| `--become`          | Ejecutar como root (sudo) |
| `-i`                | Archivo de inventario     |
| `-v`, `-vv`, `-vvv` | Verbose (más detalle)     |

---

## 5. Playbooks (Lo Importante)

Un playbook es un YAML que describe el estado deseado.

```yaml
# setup-webserver.yml
---
- name: Configurar servidor web
  hosts: webservers
  become: yes

  vars:
    app_port: 8080
    app_user: www-data

  tasks:
    - name: Actualizar cache de apt
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Instalar paquetes necesarios
      apt:
        name:
          - nginx
          - certbot
          - python3-certbot-nginx
        state: present

    - name: Crear directorio de la app
      file:
        path: /var/www/mi-app
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: "0755"

    - name: Copiar configuración de nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/mi-app
        owner: root
        group: root
        mode: "0644"
      notify: Recargar nginx

    - name: Activar sitio nginx
      file:
        src: /etc/nginx/sites-available/mi-app
        dest: /etc/nginx/sites-enabled/mi-app
        state: link
      notify: Recargar nginx

    - name: Asegurar que nginx está corriendo
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Recargar nginx
      service:
        name: nginx
        state: reloaded
```

```bash
# Ejecutar playbook
ansible-playbook -i inventory/hosts.yml setup-webserver.yml

# Dry run (ver qué haría sin hacer nada)
ansible-playbook setup-webserver.yml --check

# Ver diferencias en archivos
ansible-playbook setup-webserver.yml --check --diff

# Limitar a un servidor
ansible-playbook setup-webserver.yml --limit web1

# Con verbose
ansible-playbook setup-webserver.yml -v
```

---

## 6. Templates (Jinja2)

Los templates permiten generar archivos de configuración dinámicos.

```nginx
# templates/nginx.conf.j2
server {
    listen 80;
    server_name {{ ansible_hostname }}.ejemplo.com;

    location / {
        proxy_pass http://127.0.0.1:{{ app_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```jinja2
# templates/app-config.j2
# Configuración generada por Ansible - NO EDITAR MANUALMENTE
DATABASE_HOST={{ db_host }}
DATABASE_PORT={{ db_port | default(5432) }}
APP_ENV={{ app_env }}

{% if enable_debug %}
DEBUG=true
LOG_LEVEL=debug
{% else %}
DEBUG=false
LOG_LEVEL=info
{% endif %}

{% for server in backend_servers %}
BACKEND_{{ loop.index }}={{ server }}
{% endfor %}
```

---

## 7. Variables y Precedencia

```yaml
# En el playbook (vars:)
vars:
  app_port: 8080

# En archivos separados
# group_vars/webservers.yml  → aplica a todos los webservers
# group_vars/all.yml         → aplica a todos
# host_vars/web1.yml         → aplica solo a web1
```

```
# Estructura de directorios recomendada
proyecto-ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── host_vars/
│   └── web1.yml
├── templates/
│   └── nginx.conf.j2
├── files/
│   └── app.conf
├── playbooks/
│   ├── setup-webserver.yml
│   └── deploy-app.yml
└── roles/
    └── nginx/
```

### Ansible Vault (secrets encriptados)

```bash
# Encriptar un archivo de variables
ansible-vault create group_vars/dbservers/vault.yml

# Editar archivo encriptado
ansible-vault edit group_vars/dbservers/vault.yml

# Ejecutar playbook con vault
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Encriptar un string inline
ansible-vault encrypt_string 'mi_password_secreto' --name 'db_password'
# Resultado para pegar en el YAML:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...
```

---

## 8. Condicionales, Loops y Handlers

```yaml
tasks:
  # Condicional
  - name: Instalar paquetes Debian
    apt:
      name: nginx
    when: ansible_os_family == "Debian"

  - name: Instalar paquetes RedHat
    yum:
      name: nginx
    when: ansible_os_family == "RedHat"

  # Loop
  - name: Crear usuarios
    user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
      state: present
    loop:
      - { name: "deploy", groups: "sudo" }
      - { name: "appuser", groups: "www-data" }
      - { name: "monitor", groups: "adm" }

  # Registrar resultado
  - name: Verificar si archivo existe
    stat:
      path: /etc/app/config.conf
    register: config_file

  - name: Crear config si no existe
    template:
      src: config.conf.j2
      dest: /etc/app/config.conf
    when: not config_file.stat.exists

# Handlers (se ejecutan solo si un task notifica)
handlers:
  - name: Reiniciar nginx
    service:
      name: nginx
      state: restarted
```

---

## 9. Roles (Reutilización)

Un rol es un playbook empaquetado y reutilizable.

```bash
# Crear estructura de rol
ansible-galaxy init roles/nginx
```

```
roles/nginx/
├── defaults/
│   └── main.yml       # Variables por defecto (baja prioridad)
├── files/
│   └── nginx.conf     # Archivos estáticos
├── handlers/
│   └── main.yml       # Handlers
├── tasks/
│   └── main.yml       # Tasks principales
├── templates/
│   └── vhost.conf.j2  # Templates
└── vars/
    └── main.yml       # Variables (alta prioridad)
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Instalar nginx
  apt:
    name: nginx
    state: present

- name: Copiar config
  template:
    src: vhost.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: Recargar nginx

- name: Activar nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# Usar el rol en un playbook
---
- name: Setup webservers
  hosts: webservers
  become: yes
  roles:
    - nginx
    - certbot
    - common
```

```bash
# Instalar roles de la comunidad
ansible-galaxy install geerlingguy.docker
ansible-galaxy install -r requirements.yml
```

```yaml
# requirements.yml
roles:
  - name: geerlingguy.docker
    version: 7.1.0
  - name: geerlingguy.nginx
```

---

## 10. ansible.cfg

```ini
# ansible.cfg (en la raíz del proyecto)
[defaults]
inventory = inventory/hosts.yml
remote_user = deploy
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

---

## 11. Ejemplo Completo: Deploy de Aplicación

```yaml
# deploy-app.yml
---
- name: Deploy de la aplicación
  hosts: webservers
  become: yes

  vars:
    app_version: "v1.2.3"
    app_dir: /opt/mi-app
    app_user: appuser

  tasks:
    - name: Crear usuario de la app
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /usr/sbin/nologin

    - name: Crear directorio
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"

    - name: Descargar binario
      get_url:
        url: "https://releases.ejemplo.com/{{ app_version }}/app-linux-amd64"
        dest: "{{ app_dir }}/app"
        owner: "{{ app_user }}"
        mode: "0755"
      notify: Reiniciar app

    - name: Copiar config
      template:
        src: templates/app-config.j2
        dest: "{{ app_dir }}/config.env"
        owner: "{{ app_user }}"
        mode: "0600"
      notify: Reiniciar app

    - name: Copiar servicio systemd
      template:
        src: templates/app.service.j2
        dest: /etc/systemd/system/mi-app.service
      notify:
        - Reload systemd
        - Reiniciar app

    - name: Asegurar que la app está corriendo
      service:
        name: mi-app
        state: started
        enabled: yes

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Reiniciar app
      service:
        name: mi-app
        state: restarted
```
