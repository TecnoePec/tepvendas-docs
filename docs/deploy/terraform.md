# Terraform

## Localizacao

```
infrastructure/aws/account/tecnoepec/tepvendas/
├── ecr/main.tf              # ECR repository
├── secrets/main.tf          # Secrets Manager
├── ecs_services/
│   ├── main.tf              # Task definition + ECS service
│   ├── nlb.tf               # NLB listener porta 5050
│   ├── security_groups.tf   # SG ECS + regra Aurora
│   ├── iam.tf               # Execution + Task roles
│   └── monitoring.tf        # CloudWatch alarms + SNS
├── api_gateway/main.tf      # REST API + custom domain
├── cloudfront_web/main.tf   # CloudFront + S3
└── cicd/main.tf             # GitHub Actions OIDC policies
```

## State Backend

```hcl
backend "s3" {
  bucket         = "tecnoepec-terraform-up-and-running-state"
  key            = "tepvendas.{module}.terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "tecnoepec-terraform-up-and-running-locks"
}
```

## Tags Padrao

```hcl
default_tags = {
  terraform     = "true"
  environment   = "development"
  business_unit = "tecnoepec"
  cost_center   = "infra"
  owner         = "Juliano Leonel de Menezes"
  stack         = "tecnoepec Stack"
  account_name  = "tecnoepec"
  product       = "tepvendas"
}
```

## ECS Task Definition

- **Cluster:** main (compartilhado)
- **CPU:** 256 (0.25 vCPU)
- **Memory:** 512 MB
- **Capacity Provider:** FARGATE_SPOT
- **Container Port:** 5000
- **NLB Port:** 5050
- **Health Check:** `/health`
- **Auto-migration:** `Database.Migrate()` no startup
