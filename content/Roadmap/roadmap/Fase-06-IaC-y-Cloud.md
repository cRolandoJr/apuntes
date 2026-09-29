# Fase 6: Infrastructure as Code & Cloud — Meses 16 a 18

> **Objetivo**: Dejar de configurar infra manualmente. Todo como código, versionado, reproducible y auditable. Dominar Terraform, Ansible, y al menos un cloud provider (AWS). Preparación para Terraform Associate y AWS SAA.

---

## Mes 16: Terraform — Infraestructura como Código

### 16.1 ¿Qué es Infrastructure as Code (IaC)?

```
SIN IaC (manual):
1. Loguearme en AWS
2. Click "Create EC2"
3. Seleccionar Ubuntu, t3.micro...
4. Click "Create Security Group"
5. Agregar regla SSH...
6. Click "Launch"
Problemas: No reproducible, no versionable, errores humanos, auditoría imposible.

CON IaC (Terraform):
1. Escribir un archivo .tf describiendo lo que quiero
2. terraform apply
3. Terraform crea todo automáticamente
El archivo .tf va en Git → versionado, revisable, reproducible.
```

### 16.2 Conceptos Fundamentales de Terraform

```hcl
# main.tf — Ejemplo completo

# ==== Provider: ¿con qué cloud hablo? ====
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  # Backend: ¿dónde guardo el estado?
  backend "s3" {
    bucket = "mi-terraform-state"
    key    = "produccion/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# ==== Variables: parámetros configurables ====
variable "aws_region" {
  description = "Región de AWS"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Ambiente (staging, production)"
  type        = string
}

variable "instance_type" {
  description = "Tipo de instancia EC2"
  type        = string
  default     = "t3.micro"
}

# ==== Recursos: lo que queremos crear ====

# Red privada (VPC)
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Subnet pública
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id  # Referencia al recurso de arriba
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-subnet"
  }
}

# Security Group (firewall)
resource "aws_security_group" "api" {
  name        = "${var.environment}-api-sg"
  description = "Security group para la API"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Todo el mundo (en prod limitar esto)
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["TU_IP/32"]  # Solo tu IP puede hacer SSH
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Instancia EC2 (servidor)
resource "aws_instance" "api" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.api.id]
  key_name               = aws_key_pair.deploy.key_name

  tags = {
    Name        = "${var.environment}-api-server"
    Environment = var.environment
  }
}

# Buscar la AMI de Ubuntu más reciente
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-24.04-amd64-server-*"]
  }
}

# SSH key pair
resource "aws_key_pair" "deploy" {
  key_name   = "${var.environment}-deploy-key"
  public_key = file("~/.ssh/id_ed25519.pub")
}

# ==== Outputs: información útil post-apply ====
output "api_public_ip" {
  value       = aws_instance.api.public_ip
  description = "IP pública del servidor API"
}

output "ssh_command" {
  value = "ssh -i ~/.ssh/id_ed25519 ubuntu@${aws_instance.api.public_ip}"
}
```

### 16.3 Flujo de Trabajo de Terraform

```bash
# 1. Inicializar (descargar providers)
terraform init

# 2. Formatear código
terraform fmt

# 3. Validar sintaxis
terraform validate

# 4. Plan: ver qué VA A HACER (sin hacer nada)
terraform plan -var="environment=staging"
# Muestra: + create, ~ modify, - destroy
# SIEMPRE revisar el plan antes de apply

# 5. Aplicar: HACER los cambios
terraform apply -var="environment=staging"
# Pide confirmación. Muestra el plan de nuevo.

# 6. Ver el estado actual
terraform show
terraform state list

# 7. Destruir TODO (con cuidado!)
terraform destroy -var="environment=staging"
```

### 16.4 Terraform State

El **state file** (`terraform.tfstate`) mapea tu código a los recursos reales en la cloud. Es CRÍTICO:

- Si lo perdés, Terraform no sabe qué creó → no puede gestionar nada
- Contiene datos sensibles (IPs, IDs, a veces passwords)
- NUNCA debe ir en Git directamente
- Debe estar en un **remote backend** (S3, GCS, Terraform Cloud)
- **State locking**: Previene que dos personas apliquen cambios simultáneamente

```hcl
# Remote backend con locking
terraform {
  backend "s3" {
    bucket         = "mi-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"  # Lock via DynamoDB
    encrypt        = true              # Encriptar el state
  }
}
```

### 16.5 Módulos: Código Reutilizable

```hcl
# modules/vpc/main.tf — Módulo reutilizable para VPC
variable "environment" {}
variable "cidr_block" { default = "10.0.0.0/16" }

resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

output "vpc_id" { value = aws_vpc.this.id }

# Uso del módulo:
module "vpc_staging" {
  source      = "./modules/vpc"
  environment = "staging"
  cidr_block  = "10.0.0.0/16"
}

module "vpc_prod" {
  source      = "./modules/vpc"
  environment = "production"
  cidr_block  = "10.1.0.0/16"
}
```

**Ejercicio verificable (sin gastar plata — usa LocalStack)**:

```bash
# LocalStack emula servicios AWS localmente
docker run -d --name localstack \
  -p 4566:4566 \
  -e SERVICES=ec2,s3,dynamodb,iam \
  localstack/localstack

# Configurar Terraform para usar LocalStack
cat << 'EOF' > provider.tf
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    ec2      = "http://localhost:4566"
    s3       = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    iam      = "http://localhost:4566"
  }
}
EOF

terraform init
terraform plan
terraform apply
# Podés practicar Terraform sin gastar un centavo
```

---

## Mes 17: Ansible — Configuración como Código

### 17.1 Terraform vs Ansible

| Aspecto      | Terraform                              | Ansible                                                  |
| ------------ | -------------------------------------- | -------------------------------------------------------- |
| **Qué hace** | Crear/destruir infra (VMs, redes, DBs) | Configurar lo que ya existe (instalar software, configs) |
| **Modelo**   | Declarativo ("quiero esto")            | Procedural + Declarativo ("hacé esto en orden")          |
| **Estado**   | State file                             | Sin estado (ejecuta cada vez)                            |
| **Agente**   | No necesita agente                     | No necesita agente (usa SSH)                             |
| **Ejemplo**  | "Creá una EC2 con Ubuntu"              | "En esa EC2, instalá Docker y mi app"                    |

**Flujo típico**: Terraform crea el servidor → Ansible lo configura.

### 17.2 Conceptos de Ansible

**Inventario**: ¿En qué servidores ejecuto?

```ini
# inventory.ini
[web]
web-01 ansible_host=10.0.1.10
web-02 ansible_host=10.0.1.11

[db]
db-01 ansible_host=10.0.1.20

[production:children]
web
db

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**Playbook**: ¿Qué ejecuto?

```yaml
# setup-server.yml
---
- name: Configurar servidor de API
  hosts: web
  become: yes # Ejecutar como root (sudo)

  vars:
    app_port: 8081
    app_user: apiuser

  tasks:
    - name: Actualizar paquetes
      apt:
        update_cache: yes
        upgrade: safe

    - name: Instalar dependencias
      apt:
        name:
          - docker.io
          - docker-compose-v2
          - fail2ban
          - ufw
        state: present

    - name: Crear usuario de la aplicación
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /usr/sbin/nologin

    - name: Configurar firewall
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - "22"
        - "{{ app_port }}"

    - name: Habilitar firewall
      ufw:
        state: enabled
        default: deny

    - name: Copiar docker-compose
      copy:
        src: files/docker-compose.yml
        dest: /opt/app/docker-compose.yml
        owner: "{{ app_user }}"
        mode: "0644"

    - name: Iniciar la aplicación
      community.docker.docker_compose_v2:
        project_src: /opt/app
        state: present

    - name: Verificar que la API responde
      uri:
        url: "http://localhost:{{ app_port }}/"
        status_code: 200
      register: health
      retries: 5
      delay: 3
      until: health.status == 200
```

```bash
# Ejecutar
ansible-playbook -i inventory.ini setup-server.yml

# Verificar conectividad
ansible all -i inventory.ini -m ping

# Ejecutar un comando ad-hoc
ansible web -i inventory.ini -m shell -a "uptime"
ansible web -i inventory.ini -m shell -a "docker ps"
```

**Roles**: Playbooks organizados y reutilizables.

```
roles/
├── common/           # Configuración base para cualquier servidor
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   ├── templates/
│   └── defaults/main.yml
├── docker/           # Instalar y configurar Docker
│   ├── tasks/main.yml
│   └── handlers/main.yml
└── api/              # Deployar la API
    ├── tasks/main.yml
    ├── templates/docker-compose.yml.j2
    └── defaults/main.yml
```

---

## Mes 18: Cloud (AWS) + Proyecto Final

### 18.1 Servicios AWS Fundamentales

```
┌─────────── Compute ───────────┐
│ EC2        → Máquinas virtuales│
│ ECS/EKS   → Containers/K8s    │
│ Lambda     → Serverless        │
└───────────────────────────────┘
┌─────────── Networking ────────┐
│ VPC        → Red privada       │
│ Subnet     → Sub-redes         │
│ SG         → Firewall          │
│ ALB/NLB    → Load balancer     │
│ Route53    → DNS               │
│ CloudFront → CDN               │
└───────────────────────────────┘
┌─────────── Storage ───────────┐
│ S3         → Object storage    │
│ EBS        → Discos para EC2   │
│ EFS        → Filesystem NFS    │
└───────────────────────────────┘
┌─────────── Database ──────────┐
│ RDS        → PostgreSQL/MySQL  │
│ ElastiCache→ Redis/Memcached   │
│ DynamoDB   → Key-value NoSQL   │
└───────────────────────────────┘
┌─────────── Security ──────────┐
│ IAM        → Usuarios y permisos│
│ KMS        → Encriptación       │
│ Secrets Mgr→ Gestión de secretos│
└───────────────────────────────┘
┌─────────── Monitoring ────────┐
│ CloudWatch → Logs + métricas   │
│ CloudTrail → Auditoría de API  │
└───────────────────────────────┘
```

### 18.2 IAM (Identity and Access Management)

IAM controla QUIÉN puede hacer QUÉ en tu cuenta AWS.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "staging"
        }
      }
    }
  ]
}
```

Esto dice: "Puede ver, iniciar y detener instancias EC2, pero SOLO las que tengan el tag Environment=staging."

**Principio**: NUNCA usar la cuenta root para trabajo diario. Crear usuarios IAM con permisos mínimos.

### 18.3 VPC (Virtual Private Cloud)

```
┌──────────── VPC (10.0.0.0/16) ────────────────┐
│                                                  │
│  ┌──── AZ us-east-1a ────┐ ┌── AZ us-east-1b ─┐│
│  │ Public  10.0.1.0/24   │ │ Public 10.0.2.0/24││
│  │ ┌────┐ ┌────┐         │ │ ┌────┐            ││
│  │ │ALB │ │NAT │         │ │ │ALB │            ││
│  │ └──┬─┘ └──┬─┘         │ │ └──┬─┘            ││
│  │    │      │            │ │    │              ││
│  │ Private 10.0.10.0/24  │ │ Private 10.0.20.0/24│
│  │ ┌────┐ ┌────┐         │ │ ┌────┐ ┌────┐    ││
│  │ │App │ │App │         │ │ │App │ │App │    ││
│  │ └────┘ └────┘         │ │ └────┘ └────┘    ││
│  │                       │ │                    ││
│  │ Private 10.0.100.0/24 │ │ Private 10.0.200.0/24│
│  │ ┌─────┐               │ │ ┌─────┐           ││
│  │ │ RDS │               │ │ │ RDS │           ││
│  │ │(r/w)│               │ │ │(r/o)│           ││
│  │ └─────┘               │ │ └─────┘           ││
│  └───────────────────────┘ └────────────────────┘│
└──────────────────────────────────────────────────┘
```

- **Public subnets**: Tienen acceso directo a Internet (Internet Gateway)
- **Private subnets**: SIN acceso directo. Salen a Internet vía NAT Gateway
- **Multi-AZ**: Recursos en 2+ zonas de disponibilidad → si una AZ cae, la otra sigue

---

## Proyecto Integrador de Fase 6

### "Infraestructura Completa con Terraform + Ansible"

1. **Terraform** crea en AWS (o LocalStack):
   - VPC con subnets públicas y privadas
   - Security Groups
   - EC2 para la API
   - RDS PostgreSQL
   - S3 bucket para backups
   - Módulos reutilizables

2. **Ansible** configura los servidores:
   - Instalar Docker
   - Deployar la API con Docker Compose
   - Configurar firewall y fail2ban
   - Setup de backups automáticos a S3

3. **Todo en Git** con:
   - Estructura de directorios limpia
   - README con instrucciones
   - Variables separadas por ambiente (staging vs production)
   - CI/CD que ejecute `terraform plan` en PRs

**Verificación**:

```bash
# Terraform
terraform plan    # Sin errores
terraform apply   # Crea los recursos
terraform output  # Muestra IPs y datos

# Ansible
ansible-playbook -i inventory.ini setup.yml --check  # Dry run
ansible-playbook -i inventory.ini setup.yml           # Ejecutar

# Resultado final
curl http://$(terraform output -raw api_public_ip):8081/
# La API responde desde la cloud
```

---

## Recursos para esta Fase

### Terraform

1. **HashiCorp Learn** (developer.hashicorp.com/terraform/tutorials) — Tutoriales oficiales, excelentes
2. **"Terraform: Up & Running" de Yevgeniy Brikman** (3ra edición) — EL libro de Terraform. Práctico.
3. **Terraform Registry** (registry.terraform.io) — Documentación de providers

### Ansible

1. **Ansible Docs** (docs.ansible.com) — Bien escrita, con ejemplos
2. **"Ansible for DevOps" de Jeff Geerling** — Práctico, orientado a infra real
3. **Jeff Geerling YouTube** — Videos de Ansible, Kubernetes, homelab

### AWS

1. **AWS Free Tier** (aws.amazon.com/free) — 12 meses gratis de muchos servicios
2. **AWS Docs** (docs.aws.amazon.com) — Completa pero densa
3. **Stephane Maarek Udemy Courses** — Los mejores cursos de AWS para certificaciones
4. **LocalStack** (localstack.cloud) — Emula AWS localmente (gratis)

### Certificaciones

- **Terraform Associate** — Rendila al final del mes 18 (~$70 USD)
- **AWS Solutions Architect Associate** — Objetivo para el mes 20

---

## Checkpoint: ¿Estoy listo para la Fase 7?

- [ ] ¿Puedo crear infra con Terraform desde cero?
- [ ] ¿Puedo explicar qué es el state y por qué importa?
- [ ] ¿Puedo escribir módulos de Terraform reutilizables?
- [ ] ¿Puedo escribir un playbook de Ansible?
- [ ] ¿Puedo configurar servidores remotos con Ansible?
- [ ] ¿Puedo explicar los servicios fundamentales de AWS?
- [ ] ¿Puedo diseñar una VPC con subnets públicas y privadas?
- [ ] ¿Puedo explicar IAM y el principio de mínimo privilegio?
- [ ] ¿Puedo combinar Terraform + Ansible en un workflow?

Si respondiste 7+ de 9: avanzá a la Fase 7.
