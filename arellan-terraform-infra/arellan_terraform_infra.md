# arellan-terraform-infra — Infraestructura como Código (Terraform)

Módulos Terraform para provisionar y gestionar la infraestructura cloud del ecosistema Arellan Hnos. Aplica al entorno de producción en AWS.

## Estructura de Módulos

```
terraform/
├── environments/
│   ├── dev/
│   │   └── main.tf          # Docker Compose local (no usa Terraform)
│   ├── staging/
│   │   ├── main.tf          # Railway + Supabase (configurado manualmente)
│   │   └── variables.tf
│   └── production/
│       ├── main.tf          # AWS Full Stack
│       ├── variables.tf
│       └── outputs.tf
├── modules/
│   ├── network/             # VPC, subnets, security groups
│   ├── compute/             # EC2, ECS, ECR
│   ├── database/            # RDS PostgreSQL, backups
│   ├── cache/               # ElastiCache Redis
│   ├── storage/             # S3 buckets, CloudFront
│   ├── security/            # IAM roles, KMS, Secrets Manager
│   └── monitoring/          # CloudWatch, alarms, dashboards
└── main.tf                  # Composición de módulos
```

## Módulo `network`

```hcl
module "network" {
  source = "./modules/network"

  vpc_cidr = "10.0.0.0/16"
  environment = var.environment

  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]

  # Solo el backend y la BD están en subnets privadas
  # Los frontends (Vercel) son externos
}
```

## Módulo `database`

```hcl
module "database" {
  source = "./modules/database"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.micro"   # Producción inicial
  storage_gb     = 20
  multi_az       = false           # true cuando el volumen lo justifique

  db_name  = "arellan_production"
  username = var.db_username       # Desde AWS Secrets Manager

  backup_retention_days = 7
  backup_window         = "02:00-04:00"  # 2-4 AM Lima time

  # Encriptación en reposo obligatoria
  storage_encrypted = true
  kms_key_id        = module.security.rds_kms_key_id
}
```

## Módulo `security`

```hcl
module "security" {
  source = "./modules/security"

  # KMS keys para cifrado
  create_rds_kms_key     = true
  create_s3_kms_key      = true
  create_secrets_kms_key = true

  # IAM roles con mínimo privilegio
  ecs_task_role_permissions = [
    "s3:GetObject",
    "s3:PutObject",
    "secretsmanager:GetSecretValue",
    "cloudwatch:PutMetricData",
  ]
}
```

## Comandos de Uso

```bash
# Inicializar Terraform
terraform init

# Plan de cambios (siempre revisar antes de aplicar)
terraform plan -var-file=environments/production/terraform.tfvars

# Aplicar cambios (requiere aprobación manual del otro owner)
terraform apply -var-file=environments/production/terraform.tfvars

# Destruir entorno (SOLO en dev — nunca en producción sin consenso)
terraform destroy -var-file=environments/dev/terraform.tfvars
```

## State Management

El estado de Terraform se almacena en un bucket S3 privado con versioning y locking via DynamoDB:

```hcl
terraform {
  backend "s3" {
    bucket         = "arellan-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "arellan-terraform-locks"
  }
}
```

## Política de Cambios

Todo cambio de infraestructura de producción debe:
1. Crearse como PR en este repo
2. Pasar el pipeline de validación (`terraform validate`, `terraform plan`)
3. Ser revisado y aprobado por el segundo owner
4. Aplicarse en horario de baja actividad (después de 9 PM hora Lima)
5. Tener un plan de rollback documentado en el PR
