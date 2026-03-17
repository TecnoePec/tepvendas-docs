# CI/CD

## Backend (GitHub Actions)

**Repo:** `TecnoePec/tep_vendas_services`

**Trigger:** Push para `develop`

**Pipeline:**

1. Build & Test (.NET 10) → 765 testes
2. Security & Dependency Scan
3. Quality Gate
4. Deploy to Dev (ECS):
    - Build Docker image
    - Push para ECR (`development-tepvendas-api`)
    - Force new ECS deployment (cluster `main`, service `tepvendas-api`)

**Workflow:** `.github/workflows/dotnet-core-build.yml`

## Frontend (GitHub Actions)

**Repo:** `TecnoePec/tep_vendas_backoffice`

**Trigger:** Push para `develop`

**Pipeline:**

1. Lint & Code Quality
2. Test & Coverage → 1173 testes
3. Build (Development) → static export
4. Deploy to Dev (S3 + CloudFront):
    - Sync `out/` para S3 (`development-tepvendas-web-005200801295`)
    - Invalidate CloudFront cache (`E3NF0XZGA0HAJ5`)

**Workflow:** `.github/workflows/nextjs-build-test.yml`

**GitHub Secret necessario:** `CLOUDFRONT_DISTRIBUTION_ID` = `E3NF0XZGA0HAJ5`

## Autenticacao AWS (OIDC)

Ambos os repos usam GitHub Actions OIDC para autenticar na AWS:

- **IAM Role:** `github-actions-deploy`
- **Policies:** ECR push, ECS deploy, S3 sync, CloudFront invalidation
- **Terraform:** `infrastructure/aws/account/tecnoepec/tepvendas/cicd/main.tf`
