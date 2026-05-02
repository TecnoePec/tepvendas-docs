# Deploy AWS

## Infraestrutura

Terraform em: `infrastructure/aws/account/tecnoepec/tepvendas/`

### Modulos

| Modulo | Descricao | State Key |
|--------|-----------|-----------|
| `ecr/` | ECR repository | `tepvendas.ecr.terraform.tfstate` |
| `secrets/` | Secrets Manager (DB, JWT, Redis) | `tepvendas.secrets.terraform.tfstate` |
| `ecs_services/` | Task definition, ECS service, NLB, SG, IAM, monitoring | `tepvendas.ecs.services.terraform.tfstate` |
| `api_gateway/` | REST API, custom domain, VPC Link | `tepvendas.api.gateway.terraform.tfstate` |
| `cloudfront_web/` | CloudFront + S3 (frontend SPA) | `tepvendas.cloudfront.web.terraform.tfstate` |
| `cicd/` | CodePipeline + CodeBuild (3 pipelines) + S3 buckets de artifact | `tepvendas.cicd.terraform.tfstate` |

### Ordem de Apply

```bash
cd ecr && terraform init && terraform apply
cd ../secrets && terraform init && terraform apply
cd ../ecs_services && terraform init && terraform apply
cd ../api_gateway && terraform init && terraform apply
cd ../cloudfront_web && terraform init && terraform apply
cd ../cicd && terraform init && terraform apply
```

## Environment Variables

A aplicacao .NET usa env vars customizadas (NAO usa o padrao ASP.NET `ConnectionStrings__*`):

| Variavel | Tipo | Valor |
|----------|------|-------|
| `ASPNETCORE_ENVIRONMENT` | env | `Production` |
| `ASPNETCORE_URLS` | env | `http://+:5000` |
| `ALLOWED_ORIGINS` | env | `https://tepvendas.tecnoepec.com.br` |
| `AUTHENTICATION_ISSUER` | env | `TepVendas.API` |
| `AUTHENTICATION_AUDIENCE` | env | `TepVendas.Client` |
| `POSTGRESQL_CONNECTION` | secret | `Host=...;Port=5432;Database=tepvendas;...` |
| `AUTHENTICATION_SIGNING_KEY` | secret | JWT key (min 32 chars) |
| `Redis__ConnectionString` | secret | `endpoint:6379` |

!!! warning "Atencao"
    Os nomes das env vars sao customizados. Nao use `Jwt__Key`, `ConnectionStrings__DefaultConnection`, etc.

## IDs dos Recursos

| Recurso | ID |
|---------|-----|
| ECR | `005200801295.dkr.ecr.us-east-1.amazonaws.com/development-tepvendas-api` |
| S3 Bucket | `development-tepvendas-web-005200801295` |
| CloudFront | `E3NF0XZGA0HAJ5` |
| API Gateway | `7cso531750` |
| ECS Security Group | `sg-073c1df1810402de6` |

## Endpoints

| Endpoint | URL |
|----------|-----|
| API Gateway | `https://tepvendas-api.tecnoepec.com.br` |
| Frontend | `https://tepvendas.tecnoepec.com.br` |
| Health Check | `GET /health` |
| Swagger | `GET /swagger/v1/swagger.json` |
| API Routes | `/tepsales/v1/{resource}` |

## Recursos compartilhados (multi-projeto)

A conta TecnoePec-dev hospeda vários produtos compartilhando os mesmos recursos:

| Recurso | Compartilhado com |
|---------|-------------------|
| VPC `development-vpc` (10.10.6.0/23) | tepconfina, efatah, enersync, brandbrain, legaltech |
| ECS Cluster `main` | Todos os serviços (Fargate) |
| NLB `main` | Todos os serviços |
| Aurora PostgreSQL `tecnoepec-development` | tepconfina, tepvendas (banco `tepvendas`) |
| ElastiCache Redis `development-tecnoepec` | Todos |
| ACM `*.tecnoepec.com.br` | Todos |

!!! warning "Aurora Serverless desliga fora do horário comercial"
    O cluster Aurora é configurado com auto-stop fora do horário comercial para reduzir custo. Deploys/queries fora do horário podem falhar até o cluster reiniciar (~2 min).

!!! info "Senha do Aurora rotaciona automaticamente"
    O cluster usa `ManageMasterUserPassword`. A senha está em `arn:aws:secretsmanager:us-east-1:005200801295:secret:rds!cluster-bd261a44-...`. Quando rotaciona, o secret `development.TEPVENDAS_CONNECTION_STRING` precisa ser sincronizado manualmente (ou via Lambda automática).

## Custos

- TEP Vendas (compartilhado): **~$6/mês**
- CodeBuild (média 30 builds/mês): **~$2.60/mês**
- Textract (importadores OCR): **~$0.015 por importação**

## Monitoramento

CloudWatch Alarms (SNS → `juliano.menezes@tecnoepec.com.br`):

- ECS: Nenhuma task rodando
- ECS: CPU > 85% por 15 min
- ECS: Memoria > 85% por 15 min
- NLB: Targets unhealthy
- API: > 5 erros 5xx em 5 min
