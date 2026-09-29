[[0. Índice DevOps]]

# Cloud Fundamentals

## 1. Los 3 Grandes

| Provider  | Fortaleza                      | CLI      |
| --------- | ------------------------------ | -------- |
| **AWS**   | El más usado, más servicios    | `aws`    |
| **GCP**   | Kubernetes (GKE), data/ML      | `gcloud` |
| **Azure** | Integración Microsoft/empresas | `az`     |

> Para Platform Engineering, aprender al menos uno bien. AWS tiene la mayor demanda laboral.

---

## 2. Modelo de Responsabilidad Compartida

```
On-Premise:  VOS manejás TODO (hardware, red, OS, app)
IaaS:        El cloud da hardware/red, vos manejás OS y app (EC2, Compute Engine)
PaaS:        El cloud da hasta el runtime, vos manejás la app (App Engine, Elastic Beanstalk)
SaaS:        El cloud da todo (Gmail, Slack)
FaaS:        Solo tu código, el cloud maneja todo lo demás (Lambda, Cloud Functions)
```

---

## 3. Servicios Equivalentes

| Concepto               | AWS             | GCP               | Azure               |
| ---------------------- | --------------- | ----------------- | ------------------- |
| **VMs**                | EC2             | Compute Engine    | Virtual Machines    |
| **Kubernetes**         | EKS             | GKE               | AKS                 |
| **Contenedores**       | ECS / Fargate   | Cloud Run         | Container Instances |
| **Serverless**         | Lambda          | Cloud Functions   | Azure Functions     |
| **Object Storage**     | S3              | Cloud Storage     | Blob Storage        |
| **Block Storage**      | EBS             | Persistent Disk   | Managed Disks       |
| **SQL DB**             | RDS             | Cloud SQL         | Azure SQL           |
| **NoSQL**              | DynamoDB        | Firestore         | Cosmos DB           |
| **DNS**                | Route 53        | Cloud DNS         | Azure DNS           |
| **CDN**                | CloudFront      | Cloud CDN         | Azure CDN           |
| **Load Balancer**      | ALB / NLB       | Cloud LB          | Azure LB            |
| **VPC**                | VPC             | VPC               | VNet                |
| **IAM**                | IAM             | IAM               | Azure AD            |
| **Secrets**            | Secrets Manager | Secret Manager    | Key Vault           |
| **Container Registry** | ECR             | Artifact Registry | ACR                 |
| **CI/CD**              | CodePipeline    | Cloud Build       | Azure DevOps        |
| **Monitoreo**          | CloudWatch      | Cloud Monitoring  | Azure Monitor       |
| **Queue**              | SQS             | Pub/Sub           | Service Bus         |

---

## 4. AWS — Lo Esencial

### CLI

```bash
# Instalar
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configurar credenciales
aws configure
# AWS Access Key ID: [tu key]
# AWS Secret Access Key: [tu secret]
# Default region: us-east-1
# Default output: json

# Verificar
aws sts get-caller-identity
```

### EC2 (Servidores)

```bash
# Listar instancias
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' --output table

# Crear instancia
# ⚠️ El AMI ID es de EJEMPLO y caduca/es region-específica. Conseguí el vigente:
#   aws ec2 describe-images --owners 099720109477 \
#     --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-*-24.04-amd64-server-*" \
#     --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text
aws ec2 run-instances \
  --image-id ami-XXXXXXXX \      # reemplazar por el que devuelve el comando de arriba
  --instance-type t3.micro \
  --key-name mi-key \
  --security-group-ids sg-xxx \
  --subnet-id subnet-xxx

# Parar / iniciar
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 start-instances --instance-ids i-xxx
```

### S3 (Storage)

```bash
# Crear bucket
aws s3 mb s3://mi-bucket-unico

# Subir archivo
aws s3 cp archivo.txt s3://mi-bucket/
aws s3 cp ./carpeta/ s3://mi-bucket/carpeta/ --recursive

# Listar
aws s3 ls s3://mi-bucket/

# Descargar
aws s3 cp s3://mi-bucket/archivo.txt ./

# Sincronizar (como rsync)
aws s3 sync ./local/ s3://mi-bucket/

# Borrar
aws s3 rm s3://mi-bucket/archivo.txt
aws s3 rb s3://mi-bucket --force    # Borrar bucket completo
```

### IAM (Permisos)

```bash
# Listar usuarios
aws iam list-users

# Crear usuario
aws iam create-user --user-name deploy-user

# Crear access key
aws iam create-access-key --user-name deploy-user

# Adjuntar policy
aws iam attach-user-policy --user-name deploy-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### EKS (Kubernetes Managed)

```bash
# Crear cluster con eksctl
eksctl create cluster \
  --name mi-cluster \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3

# Configurar kubectl
aws eks update-kubeconfig --name mi-cluster --region us-east-1

# Verificar
kubectl get nodes
```

---

## 5. Networking en Cloud

```
VPC (Virtual Private Cloud):
  Tu red privada en el cloud

Subnets:
  ├── Pública  (tiene Internet Gateway, acceso directo a internet)
  └── Privada  (sin acceso directo, necesita NAT Gateway para salir)

Security Groups:
  Firewall por instancia (stateful)

NACL (Network ACL):
  Firewall por subnet (stateless)

Diseño típico:
  VPC 10.0.0.0/16
  ├── Subnet pública  10.0.1.0/24  (Load Balancer, Bastion)
  ├── Subnet pública  10.0.2.0/24  (otra AZ)
  ├── Subnet privada  10.0.10.0/24 (App servers)
  ├── Subnet privada  10.0.11.0/24 (otra AZ)
  ├── Subnet privada  10.0.20.0/24 (Base de datos)
  └── Subnet privada  10.0.21.0/24 (otra AZ)
```

---

## 6. Conceptos Importantes

### Availability Zones (AZ)

```
Una región (ej: us-east-1) tiene múltiples AZs (data centers):
  us-east-1a, us-east-1b, us-east-1c

Regla: Distribuir recursos en al menos 2 AZs para alta disponibilidad
```

### Tags

```
Tagear TODO. Es cómo organizás y controlás costos.
  Name: web-server-prod
  Environment: production
  Team: platform
  ManagedBy: terraform
```

### Costos

```
1. Instancias EC2: por hora/segundo
2. Storage S3: por GB almacenado + requests
3. Data Transfer: salida del cloud cobra, entrada gratis
4. Load Balancers: por hora + datos procesados
5. NAT Gateway: por hora + GB procesado (caro si hay mucho tráfico)

Tips:
  - Usar instancias Reserved (1-3 años) para prod: -40% a -70%
  - Usar Spot Instances para workloads tolerantes a interrupciones: -90%
  - Apagar recursos de dev/staging fuera de horario laboral
  - Alertas de facturación (AWS Budgets)
```

---

## 7. Terraform + Cloud = El Combo

```hcl
# Ejemplo: VPC + EC2 + Security Group en AWS con Terraform
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags = { Name = "public-1a" }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id   # data source, no hardcodear (ver Terraform.md §7)
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  tags = {
    Name        = "web-server"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}
```

> La idea es que NUNCA crees recursos desde la consola web en producción. Siempre Terraform.
