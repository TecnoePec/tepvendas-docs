# TepVendas - Documentacao Tecnica

Sistema de gestao de vendas e pedidos para a Major Nutricao Animal, desenvolvido pela TecnoePec.

## Repositorios

| Repositorio | Descricao | Stack |
|-------------|-----------|-------|
| [tepvendas-api](https://github.com/TecnoePec/tep_vendas_services) | Backend API | .NET 10, PostgreSQL, Redis |
| [tepvendas-web](https://github.com/TecnoePec/tep_vendas_backoffice) | Backoffice Web | Next.js 14, React 18, TypeScript |
| [tepvendas-mobile](https://github.com/TecnoePec/tep_vendas_mobile) | App Mobile | Flutter 3, Dart |
| [tepvendas-docs](https://github.com/TecnoePec/tepvendas-docs) | Documentacao | MkDocs Material |

## Ambientes

| Ambiente | Backend API | Frontend Web |
|----------|-------------|--------------|
| Production | `https://tepvendas-api.tecnoepec.com.br` | `https://tepvendas.tecnoepec.com.br` |
| Local | `http://localhost:5001` | `http://localhost:3000` |

## Quick Start

```bash
# Backend
cd tep_vendas_services
docker-compose up -d
# API disponivel em http://localhost:5001
# Swagger em http://localhost:5001/swagger

# Frontend
cd tep_vendas_backoffice
npm install && npm run dev
# Web disponivel em http://localhost:3000
```

## Stack Tecnologico

- **Backend**: ASP.NET Core 10, Entity Framework Core, PostgreSQL 17, Redis 7
- **Frontend**: Next.js 14, React 18, Material-UI, TypeScript 5
- **Mobile**: Flutter 3, Dart, Riverpod, Firebase
- **Infra**: AWS ECS Fargate, Aurora PostgreSQL, ElastiCache Redis, API Gateway, CloudFront + S3
- **CI/CD**: GitHub Actions, ECR, OIDC
- **Observabilidade**: Sentry, CloudWatch, SNS Alarms
