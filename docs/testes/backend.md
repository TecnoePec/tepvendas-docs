# Testes — Backend (.NET 10)

Suite de testes automatizada do `tep_vendas_services`, executada localmente e no CI (CodeBuild).

## Estado atual

| Métrica | Valor |
|---|---|
| **Total** | 796 testes passando |
| **Ignorados** | 2 (`Pay_AfterApproval_Returns200_AndSetsPaidStatus`, `Pay_TwiceIsIdempotent`) |
| **Falhas** | 0 |
| **Framework** | NUnit 3 + Moq + Coverlet |
| **Reporter CI** | JUnitTestLogger + Code Coverage Summary (Cobertura XML) |

## Distribuição

| Projeto | Testes | Escopo |
|---|---|---|
| `Tep.Sales.Service.Service.Tests` | 517 | Serviços (regra de negócio), validators, helpers |
| `Tep.Sales.Service.Host.Tests` | 279 (+2 skipped) | Controllers HTTP, mappers, security filters |

## Como rodar

**Toda a suite:**
```bash
cd tep_vendas_services
dotnet test src/Tep.Sales.Service.sln
```

**Só um projeto:**
```bash
dotnet test src/Tep.Sales.Service.Service.Tests/
```

**Com filtro:**
```bash
dotnet test --filter "FullyQualifiedName~PaymentCondition"
dotnet test --filter "TestCategory=Integration"
```

**Coverage:**
```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

Relatório em `TestResults/*/coverage.opencover.xml`.

## No CI (CodeBuild)

Roda no stage BUILD do `buildspec.yml`:

```yaml
- dotnet test src/Tep.Sales.Service.sln --logger "junit;LogFilePath=test-results/{assembly}.xml"
```

Artefatos publicados: `test-results/*.xml` + `coverage.opencover.xml`. Falha na suite bloqueia o deploy.

## Convenções

- **Arrange / Act / Assert** — três comentários explícitos em cada teste.
- **Nomes:** `MethodName_Scenario_ExpectedBehavior` (ex: `Execute_WithInvalidEmail_ReturnsEmpty`).
- **Base classes:** `ServiceBaseTest` (Services) e `ControllerBaseTest` (Host) provisionam `AppDbContext` in-memory + `IMapper` + mocks comuns.
- **Mocks:** `Moq` — sempre `Strict` para forçar setup explícito.
- **Multi-tenancy:** cada teste roda com `CompanyId` fixo (`TestDataFactory.CompanyId`); o Global Query Filter é respeitado.

## Áreas com cobertura relevante

- **Autenticação:** `SignInService`, `RefreshTokenService`, `PasswordHashService` (bcrypt + AES legacy).
- **Wizard/Orçamento:** `AddPurchaseOrderService`, `UpdatePurchaseOrderService`, validators de itens/frete.
- **Comissão hierárquica (RTV):** `CommissionCalculationService` com bonificação de 30t sobre volume da equipe.
- **Descontos:** `DiscountRuleService`, `DiscountWeightService`, resolução por produto vs. linha vs. peso.
- **Audit interceptor:** `AuditSaveChangesInterceptor` gera `AuditLog` automaticamente em SaveChanges (exceto para AuditLog, Notification, PushToken, RefreshToken).

## Testes ignorados

Os 2 skipped no `Host.Tests` são de `PurchaseOrderPaymentController` — cenários de gateway externo que ainda não têm mock estável. Não bloqueia deploy.

## Troubleshooting

**"The database operation was expected to affect 1 row(s), but actually affected 0"** — teste rodou em paralelo com outro que alterou a row. Adiciona `[NonParallelizable]` no teste ou usa `AsNoTracking()`.

**Coverage não aparece no CI** — checa se `coverlet.msbuild` está referenciado no `.csproj` do projeto de teste e se o path `**/coverage.opencover.xml` bate com o glob no CodeBuild.
