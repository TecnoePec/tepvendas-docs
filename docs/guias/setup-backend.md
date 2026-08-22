# Setup Backend

Ambiente para rodar o **`tep_vendas_services`** (.NET 10 / EF Core / PostgreSQL) localmente.

## Pré-requisitos

- [.NET SDK 10.0.x](https://dotnet.microsoft.com/download/dotnet/10.0)
- **PostgreSQL 15+** rodando local, ou acesso ao Aurora dev via VPN
- (Opcional) **Redis** local pra testes de cache
- **AWS CLI** configurado com o profile `tecnoepec-dev` pra ler secrets

## Clonar + Restore

```bash
git clone git@github.com:TecnoePec/tep_vendas_services.git
cd tep_vendas_services
dotnet restore src/Tep.Sales.Service.sln
```

## Connection string

Backend lê `POSTGRESQL_CONNECTION`. Duas opções:

**A) Aurora dev (via VPN):**

```bash
export POSTGRESQL_CONNECTION="$(aws secretsmanager get-secret-value \
    --secret-id development.TEPVENDAS_CONNECTION_STRING \
    --profile tecnoepec-dev --region us-east-1 \
    --query SecretString --output text)"
```

**B) Postgres local:**

```bash
export POSTGRESQL_CONNECTION="Host=localhost;Port=5432;Database=tepvendas;Username=postgres;Password=postgres"
```

Para local, cria o banco:

```bash
createdb -U postgres tepvendas
```

## Rodar migrations + seed

```bash
cd src/Tep.Sales.Service.Host
dotnet ef database update \
    --project ../Tep.Sales.Service.Data \
    --context AppDbContext
```

Ao iniciar o Host (`dotnet run`), o `SeedTep.ExecuteAsync()` popula automaticamente:

- 2 Companies + 1 usuário dev (`dev@tep.com.br` / senha inicial pelo `SeedTep`)
- 1 CompanyGlobalParameter por Company (defaults)
- Catálogo completo (`SeedCatalog`): 6 ProductLines + 6 ProductGroups + 8 PaymentConditions + 2 PriceTables + 16 PaymentPriceTables + 31 Products + 447 PriceTableItems
- Fixture do wizard (`SeedFixtures`): 1 DC + 1 Client + 2 addresses + 1 DCCA
- Descontos e comissões (`SeedDiscountsAndCommissions`)

Todos idempotentes — pode rodar múltiplas vezes.

Detalhes: [Deploy/Migrations](../deploy/migrations.md).

## Rodar o Host

```bash
dotnet run --project src/Tep.Sales.Service.Host
```

Swagger em <http://localhost:5164/swagger>.

## Testes

```bash
dotnet test src/Tep.Sales.Service.sln
```

Suite atual: **796 testes** (517 Service.Tests + 279 Host.Tests + 2 skipped).

## Secrets locais

Cria `src/Tep.Sales.Service.Host/appsettings.Development.json` (não versionado):

```json
{
  "Jwt": { "SecretKey": "<qualquer-string-de-32+-chars>" },
  "Redis": { "ConnectionString": "localhost:6379" }
}
```

Alternativa: exportar variáveis de ambiente equivalentes (`Jwt__SecretKey`, `Redis__ConnectionString`) — o `IConfiguration` do ASP.NET pega automaticamente.

## Troubleshooting

**Erro `The database operation was expected to affect 1 row(s), but actually affected 0`:** concorrência EF Core. Provavelmente alguém alterou a mesma row via SQL direto. Refaz a query com `.AsNoTracking()` antes do Update ou usa `PATCH` em vez de `PUT`.

**Aurora "timeout expired" no macOS:** você não está conectado na VPN, ou a senha rotacionou. `aws secretsmanager get-secret-value` para pegar a nova.

**ECS retorna HTTP 500 sem log:** o Aurora escala pra zero fora do horário comercial. `aws ecs update-service --desired-count 1` acorda o serviço.
