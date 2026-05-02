# Observabilidade

## Stack atual

| Camada | Ferramenta | Detalhes |
|--------|------------|----------|
| **Logs (backend)** | CloudWatch Logs | Log group `/ecs/development-tepvendas-api` |
| **Logs (CodeBuild)** | CloudWatch Logs | Log group `/aws/codebuild/development-tepvendas-{api\|web\|mobile}-build` |
| **Métricas** | CloudWatch Metrics | ECS, NLB, RDS, ElastiCache (built-in) |
| **Alarmes** | CloudWatch Alarms + SNS | E-mail para `juliano.menezes@tecnoepec.com.br` |
| **Health check (API)** | Endpoint `/health` | Verifica PostgreSQL e Redis |
| **Erros mobile** | Firebase Crashlytics | Stack traces de Android/iOS |
| **Analytics mobile** | Firebase Analytics | Eventos de uso |

!!! info "Sentry foi removido em mar/2026"
    O Sentry estava configurado em backend, frontend e mobile. Foi removido para reduzir dependências externas. A planejamento é substituir por um serviço **OpenTelemetry** centralizado que vai atender todos os projetos da TecnoePec.

## Logs do Backend

### Acessar logs em tempo real

```bash
aws logs tail /ecs/development-tepvendas-api \
    --follow \
    --profile tecnoepec-dev \
    --region us-east-1
```

### Buscar erros recentes

```bash
aws logs filter-log-events \
    --log-group-name /ecs/development-tepvendas-api \
    --filter-pattern "Exception" \
    --start-time $(date -v-30M +%s000) \
    --profile tecnoepec-dev \
    --region us-east-1
```

### Estrutura do log

Cada log line inclui (formato `Tep.Libraries.Logging`):

```
[Trace: <trace-id>] - [Span: <span-id>] - [Request: <req-id>] - [Session: <session-id>] - [<source>] : <message>
```

## Health Check

```http
GET https://tepvendas-api.tecnoepec.com.br/health
```

Resposta:

```json
{
  "status": "Healthy",
  "checks": [
    { "name": "postgres", "status": "Healthy" },
    { "name": "redis", "status": "Healthy" }
  ]
}
```

O health check do Redis é configurado como **Degraded** em caso de falha (não derruba o `/health` inteiro), porque o Redis é usado apenas para DataProtection (chaves JWT) e cache, e a aplicação pode operar sem ele.

## Alarmes Ativos

Configurados via Terraform em `infrastructure/aws/account/tecnoepec/tepvendas/ecs_services/`:

| Alarme | Threshold | Ação |
|--------|-----------|------|
| ECS: Nenhuma task rodando | `RunningTaskCount = 0` por 1 min | SNS → e-mail |
| ECS: CPU alto | `CPUUtilization > 85%` por 15 min | SNS → e-mail |
| ECS: Memória alta | `MemoryUtilization > 85%` por 15 min | SNS → e-mail |
| NLB: Targets unhealthy | `UnHealthyHostCount > 0` por 2 min | SNS → e-mail |
| API: Erros 5xx | `> 5 erros 5xx em 5 min` | SNS → e-mail |

## Mobile (Firebase)

### Crashlytics

Console: https://console.firebase.google.com/project/tepvendas-mobile/crashlytics

Crashes são reportados automaticamente quando o app sobe. Stack traces vêm desofuscados se o build incluir o mapping file.

### Analytics

Console: https://console.firebase.google.com/project/tepvendas-mobile/analytics

Eventos custom rastreados:
- `login`
- `purchase_order_created`
- `sync_started` / `sync_completed`
- `pdf_export`

## Dashboards CloudWatch

Dashboard `tepvendas-development` (a ser criado/migrado para Grafana).

## Próximos passos (planejado)

- **OpenTelemetry centralizado** — substituir o Sentry com um serviço próprio que receba traces/metrics/logs de todos os projetos TecnoePec
- **Grafana** — dashboards unificados sobre Prometheus/CloudWatch
- **Cost Allocation Tags** — ativar na conta master para visibilidade de custo por projeto
