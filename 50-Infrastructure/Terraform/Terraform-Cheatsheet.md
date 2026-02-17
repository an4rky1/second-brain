---
created: 2026-02-16
tags:
  - cheat-sheet
  - terraform
  - iac
  - devops
  - cloud
aliases:
  - Terraform Cheatsheet
  - Terraform Reference
related:
  - Kubernetes-Cheatsheet
  - AWS-Basics
  - Cloud-Design-Patterns
---

# Terraform — Полная шпаргалка

> [!SUMMARY] Обзор
> Terraform — инструмент Infrastructure as Code (IaC) от HashiCorp. Декларативное описание инфраструктуры, управление состоянием, модульность. Поддерживает 100+ провайдеров.

---

## 📚 Теория

### Terraform Workflow

```
┌─────────────────────────────────────────────────────┐
│  1. Write   →  main.tf (описание инфраструктуры)    │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  2. Init    →  terraform init (инициализация)       │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  3. Plan    →  terraform plan (предпросмотр)        │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  4. Apply   →  terraform apply (применение)         │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  5. State   →  terraform.tfstate (состояние)        │
└─────────────────────────────────────────────────────┘
```

### State Management

```
┌─────────────────────────────────────────────────────┐
│              Remote State (рекомендуется)            │
│  ┌─────────────────────────────────────────────┐    │
│  │  S3 + DynamoDB (AWS)                        │    │
│  │  Azure Blob Storage                         │    │
│  │  GCS (Google Cloud)                         │    │
│  │  Terraform Cloud/Enterprise                 │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## ⚡ Быстрый старт

```bash
# Установка
brew install terraform
# или
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# Проверка
terraform version

# Инициализация
terraform init

# Планирование
terraform plan
terraform plan -out=tfplan
terraform plan -var="instance_type=t2.micro"

# Применение
terraform apply
terraform apply tfplan
terraform apply -auto-approve

# Уничтожение
terraform destroy
terraform destroy -auto-approve

# State
terraform state list
terraform state show resource.name
terraform state mv old.name new.name
terraform state rm resource.name
terraform import resource.name resource_id
```

---

## 🔧 Практические примеры

### Базовая конфигурация

```hcl
# main.tf

# Провайдер
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  
  # Remote state (рекомендуется)
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Конфигурация провайдера
provider "aws" {
  region = "us-east-1"
  
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
      Project     = "myapp"
    }
  }
}

# Переменные
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
  
  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Instance type must be t2.* or t3.*"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}

# Locals
locals {
  common_tags = merge(
    {
      Environment = var.environment
      ManagedBy   = "terraform"
    },
    var.tags
  )
  
  name_prefix = "myapp-${var.environment}"
}

# Data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  owners = ["099720109477"] # Canonical
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "all" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Resources
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Security group for web servers"
  vpc_id      = data.aws_vpc.default.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"] # Только из внутренней сети
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = local.common_tags
}

resource "aws_instance" "web" {
  count         = 2
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  
  subnet_id              = element(data.aws_subnets.all.ids, count.index)
  vpc_security_group_ids = [aws_security_group.web.id]
  
  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }
  
  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y nginx
              systemctl start nginx
              echo "Hello from ${local.name_prefix}-${count.index}" > /var/www/html/index.html
              EOF
  
  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web-${count.index}"
      Role = "web"
    }
  )
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags]
  }
}

# Outputs
output "instance_ids" {
  description = "IDs of EC2 instances"
  value       = aws_instance.web[*].id
}

output "instance_public_ips" {
  description = "Public IPs of EC2 instances"
  value       = aws_instance.web[*].public_ip
}

output "security_group_id" {
  description = "ID of security group"
  value       = aws_security_group.web.id
}
```

### Modules

```hcl
# modules/vpc/main.tf
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "environment" {
  type    = string
}

variable "availability_zones" {
  type    = list(string)
}

locals {
  name_prefix = "vpc-${var.environment}"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = local.name_prefix
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name        = "${local.name_prefix}-private-${count.index + 1}"
    Environment = var.environment
    Type        = "private"
  }
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name        = "${local.name_prefix}-public-${count.index + 1}"
    Environment = var.environment
    Type        = "public"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name        = local.name_prefix
    Environment = var.environment
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

```hcl
# Использование модуля
module "vpc" {
  source = "./modules/vpc"
  # или
  # source  = "terraform-aws-modules/vpc/aws"
  # version = "5.0.0"
  
  vpc_cidr           = "10.0.0.0/16"
  environment        = var.environment
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  tags = {
    Project = "myapp"
  }
}

# Выходные значения модуля
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

### Workspaces

```bash
# Создание workspace
terraform workspace new production
terraform workspace new staging
terraform workspace new development

# Переключение
terraform workspace select production
terraform workspace show

# List
terraform workspace list

# Использование в конфиге
resource "aws_instance" "example" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = terraform.workspace
  }
}
```

### State Management

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "env:/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    
    # Для cross-account
    # role_arn = "arn:aws:iam::ACCOUNT_ID:role/TerraformRole"
  }
}

# backend "azurerm" {
#   resource_group_name  = "terraform-state-rg"
#   storage_account_name = "tfstate"
#   container_name       = "tfstate"
#   key                  = "production.terraform.tfstate"
# }

# backend "gcs" {
#   bucket = "my-terraform-state"
#   prefix = "terraform/state"
# }
```

```bash
# Импорт существующих ресурсов
terraform import aws_instance.example i-1234567890abcdef0

# Импорт модуля
terraform import module.vpc.aws_vpc.main vpc-1234567890abcdef0

# Перемещение ресурсов
terraform state mv aws_instance.old aws_instance.new
terraform state mv module.old.aws_instance.aws_instance.new

# Удаление из state (не удаляет ресурс!)
terraform state rm aws_instance.example

# Показать ресурс
terraform state show aws_instance.example
```

### Providers

```hcl
# Multiple providers
provider "aws" {
  alias  = "us-east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us-west"
  region = "us-west-2"
}

resource "aws_instance" "east" {
  provider      = aws.us-east
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}

resource "aws_instance" "west" {
  provider      = aws.us-west
  ami           = "ami-87654321"
  instance_type = "t3.micro"
}

# Google Cloud
provider "google" {
  project = "my-project"
  region  = "us-central1"
}

# Azure
provider "azurerm" {
  features {}
}

# Kubernetes
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "production"
}

# Helm
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}
```

---

## 🎯 Best Practices

### ✅ Делать

```hcl
# 1. Remote state
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# 2. Version pinning
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 3. Modules для повторного использования
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# 4. Tags
provider "aws" {
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# 5. Sensible defaults
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

# 6. Validation
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### ❌ Не делать

```hcl
# 1. Хардкод секретов
resource "aws_db_instance" "default" {
  password = "supersecret"  # ❌
  
  # Используйте
  password = var.db_password  # ✅
}

# 2. Local state в production
# backend "local" {}  # ❌

# 3. Без версионирования
provider "aws" {}  # ❌
provider "aws" {
  version = "~> 5.0"  # ✅
}

# 4. Огромные монолитные конфиги
# Разделяйте на модули и workspace'ы

# 5. Игнорирование плана
# Всегда делайте terraform plan перед apply
```

---

## 🐛 Troubleshooting

| Проблема | Решение |
|----------|---------|
| `State lock` | `terraform force-unlock LOCK_ID` |
| `Provider not found` | `terraform init -upgrade` |
| `Resource not found` | `terraform import` или обновите state |
| `Drift detected` | `terraform refresh` или `terraform apply` |
| `Backend config changed` | `terraform init -migrate-state` |

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — K8s инфраструктура
- [[AWS-Basics]] — AWS основы
- [[Cloud-Design-Patterns]] — Паттерны облака

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Remote state** — обязательно для команд
> 2. **State locking** — предотвращает конфликты
> 3. **Modules** — DRY для инфраструктуры
> 4. **Workspaces** — изоляция окружений
> 5. **Plan before apply** — всегда проверяйте изменения

> [!INFO] Полезные команды
> ```bash
> # Форматирование
> terraform fmt -recursive
> 
> # Валидация
> terraform validate
> 
> # Граф зависимостей
> terraform graph | dot -Tsvg > graph.svg
> 
> # Очистка
> terraform state rm resource  # Удалить из state
> 
> # Import
> terraform import resource.id resource_id
> 
> # Workspace
> terraform workspace list
> terraform workspace new prod
> ```
