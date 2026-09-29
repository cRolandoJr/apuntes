[[0. Índice DevOps]]

# Terraform — Infrastructure as Code

## 1. ¿Qué es Terraform?

Terraform te permite **definir infraestructura como código**. En vez de crear servidores, redes y bases de datos manualmente desde una consola web, lo escribís en archivos `.tf` y Terraform los crea/modifica/destruye por vos.

```
Ansible = CONFIGURACIÓN de servidores (qué software instalar, qué config poner)
Terraform = CREACIÓN de infraestructura (servidores, redes, DNS, bases de datos)
```

- **Declarativo:** Describís el estado final deseado
- **Idempotente:** Si ya existe, no lo recrea
- **Multi-cloud:** AWS, GCP, Azure, DigitalOcean, etc.
- **State:** Mantiene un archivo de estado para saber qué existe

---

## 2. Instalación

```bash
# Descargar desde releases oficiales
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform --version

# O con apt (Ubuntu/Debian)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

---

## 3. Estructura de un Proyecto

```
mi-infra/
├── main.tf           # Recursos principales
├── variables.tf      # Declaración de variables
├── outputs.tf        # Valores de salida
├── terraform.tfvars  # Valores de las variables (NO commitear si tiene secrets)
├── providers.tf      # Configuración de providers
├── versions.tf       # Versiones requeridas
└── .terraform/       # Directorio interno (no commitear)
```

---

## 4. Conceptos Básicos

### Provider (¿Dónde crear la infra?)

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### Resources (¿Qué crear?)

```hcl
# main.tf

# Crear un servidor (instancia EC2 en AWS)
resource "aws_instance" "web" {
  # ⚠️ NUNCA hardcodear la AMI (es region-específica y caduca → "InvalidAMIID").
  # Usar un data source que resuelve la última Ubuntu (ver sección 7): ami = data.aws_ami.ubuntu.id
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Crear un Security Group (firewall)
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Permitir HTTP y SSH"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.mi_ip]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Variables

```hcl
# variables.tf
variable "aws_region" {
  description = "Región de AWS"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "Tipo de instancia EC2"
  type        = string
  default     = "t3.micro"
}

variable "mi_ip" {
  description = "Mi IP para acceso SSH"
  type        = string
  # Sin default = obligatorio pasarlo
}

variable "environment" {
  description = "Ambiente"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment debe ser dev, staging o production."
  }
}
```

```hcl
# terraform.tfvars (valores)
aws_region    = "us-east-1"
instance_type = "t3.small"
mi_ip         = "203.0.113.50/32"
environment   = "production"
```

### Outputs

```hcl
# outputs.tf
output "server_ip" {
  description = "IP pública del servidor"
  value       = aws_instance.web.public_ip
}

output "server_dns" {
  description = "DNS público"
  value       = aws_instance.web.public_dns
}
```

---

## 5. Workflow (Comandos)

```bash
# 1. Inicializar (descargar providers)
terraform init

# 2. Validar sintaxis
terraform validate

# 3. Formatear código
terraform fmt
terraform fmt -recursive

# 4. Plan (ver qué va a hacer SIN hacer nada)
terraform plan
terraform plan -out=plan.tfplan    # Guardar plan

# 5. Aplicar (crear/modificar la infra)
terraform apply
terraform apply plan.tfplan        # Aplicar plan guardado
terraform apply -auto-approve      # Sin confirmación (solo en CI/CD)

# 6. Ver estado actual
terraform show
terraform state list               # Listar recursos
terraform state show aws_instance.web   # Detalle de un recurso

# 7. Destruir TODO
terraform destroy

# 8. Destruir un recurso específico
terraform destroy -target=aws_instance.web
```

```
terraform plan output:
  + = crear
  ~ = modificar
  - = destruir
  -/+ = destruir y recrear
```

---

## 6. State (Estado)

Terraform guarda un archivo `terraform.tfstate` que mapea tu código a la realidad.

```bash
# Ver estado
terraform state list
terraform state show aws_instance.web

# Mover un recurso (renombrar)
terraform state mv aws_instance.web aws_instance.webserver

# Importar un recurso existente (creado manualmente)
terraform import aws_instance.web i-1234567890abcdef0

# Eliminar del estado (sin destruir el recurso real)
terraform state rm aws_instance.web
```

### Remote State (para equipos)

```hcl
# versions.tf — Guardar estado en S3 (compartido)
terraform {
  backend "s3" {
    bucket         = "mi-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"    # Locking para evitar conflictos
    encrypt        = true
  }
}
```

> **Regla:** NUNCA commitear `terraform.tfstate` a git. Contiene datos sensibles. Usar remote state.

---

## 7. Data Sources (Leer datos existentes)

```hcl
# Buscar la AMI más reciente de Ubuntu
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]    # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# Usarla
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
}

# Leer VPC existente
data "aws_vpc" "default" {
  default = true
}
```

---

## 8. Módulos (Reutilización)

```hcl
# Estructura
modules/
└── webserver/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf

# modules/webserver/main.tf
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name = var.name
  }
}

# modules/webserver/variables.tf
variable "ami_id" { type = string }
variable "instance_type" { type = string; default = "t3.micro" }
variable "name" { type = string }

# modules/webserver/outputs.tf
output "public_ip" { value = aws_instance.this.public_ip }
```

```hcl
# Usar el módulo en main.tf
module "web_prod" {
  source        = "./modules/webserver"
  ami_id        = data.aws_ami.ubuntu.id
  instance_type = "t3.small"
  name          = "web-prod"
}

module "web_staging" {
  source        = "./modules/webserver"
  ami_id        = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  name          = "web-staging"
}

# Usar módulos de la comunidad (Terraform Registry)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

---

## 9. Loops y Condicionales

```hcl
# count — crear N recursos
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index + 1}"
  }
}

# for_each — crear recursos desde un mapa
variable "servers" {
  default = {
    web    = "t3.small"
    api    = "t3.medium"
    worker = "t3.micro"
  }
}

resource "aws_instance" "servers" {
  for_each      = var.servers
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value

  tags = {
    Name = each.key
  }
}

# Condicional
resource "aws_instance" "bastion" {
  count         = var.environment == "production" ? 1 : 0
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```

---

## 10. Buenas Prácticas

```
1. Siempre hacer `terraform plan` antes de `apply`
2. Usar remote state con locking
3. No commitear .tfstate ni .tfvars con secrets
4. Usar módulos para código reutilizable
5. Versionar providers y módulos
6. Tagear TODO con Name, Environment, ManagedBy
7. Usar workspaces para separar ambientes (o directorios separados)
8. Revisar el plan en code review (PR)
```

### .gitignore para Terraform

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars       # Si contiene secrets
crash.log
*.tfplan
```

> ⚠️ **NO ignores `.terraform.lock.hcl` — hay que COMMITEARLO.** Es el lock de
> versiones de providers; commitearlo garantiza que vos, el equipo y el CI resuelvan
> exactamente las mismas versiones (igual que `package-lock.json` o `flake.lock`).
> Ignorarlo = cada `init` puede traer versiones distintas. (Guía oficial de HashiCorp.)
