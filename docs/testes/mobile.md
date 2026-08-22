# Testes — Mobile (Flutter 3.38)

Suite de testes automatizada do `tep_vendas_mobile`, executada localmente e no CI (CodeBuild).

## Estado atual

| Métrica | Valor |
|---|---|
| **Total** | 871 testes passando |
| **Falhas** | 0 |
| **Framework** | `flutter_test` + `mockito` |
| **Duração local** | ~30s |

## Como rodar

**Toda a suite:**
```bash
cd tep_vendas_mobile
flutter test
```

**Só um arquivo:**
```bash
flutter test test/services/audit/load_audit_test.dart
```

**Só um teste por nome:**
```bash
flutter test --plain-name "renders 5 ElevatedButton"
```

**Coverage:**
```bash
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html   # opcional
```

## Estrutura

```
test/
├── app/                    # Testes de app.dart / bootstrap
├── components/             # Widgets reutilizáveis (TepMultiStepForm etc)
├── domain/                 # Entities e DTOs
├── integration/            # Fluxos end-to-end (sync, wizard)
├── models/errors/          # AppBaseException e derivadas
├── repositories/           # BaseRepository, wrappers Hive
├── services/
│   ├── audit/              # LoadAudit, SyncAudit
│   ├── auth/               # AuthService, RefreshTokenService
│   ├── price_table/        # PriceTableService
│   ├── purchase_order/     # Orçamento (cálculo, descontos, comissão)
│   └── sync/               # SynchronizationDataService + local_to_remote/*
├── utils/                  # Helpers, formatters, globals
└── widgets/                # Páginas (client, product, price_table, ...)
```

## No CI (CodeBuild)

Roda no stage BUILD do `buildspec.yml`:

```yaml
- flutter analyze --no-fatal-infos --no-fatal-warnings
- flutter test
- flutter build apk --release --dart-define=ENV=production
```

O `flutter test` sem `|| true` — falha bloqueia o build. `flutter analyze` roda com `--no-fatal` porque tem warnings pré-existentes de null-aware unnecessary que ainda não foram limpos (baixa prioridade).

## Convenções

- **Nomes de teste:** frase em pt-BR descrevendo comportamento (ex: `"SyncClient.execute reporta progresso via callback"`).
- **`group('X', () { … })`** em volta de cada classe/serviço testado.
- **Mocks com `mockito`** — arquivo `.mocks.dart` gerado via `flutter pub run build_runner build`.
- **Finders estáveis para widgets:** `find.byType(SomeWidget)` faz match ESTRITO — pra pegar subtipos (ex: `ElevatedButton.icon` retorna `_ElevatedButtonWithIcon`), usar `find.byWidgetPredicate((w) => w is ElevatedButton)`. Nunca depender de nome de classe privado do SDK (`_ElevatedButtonWithIcon`) — muda entre versões do Flutter e quebra silenciosamente.

## Áreas com cobertura relevante

- **Sync engine:** `SyncAudit`, `SynchronizationDataService`, `local_to_remote/*`, `entity_creator` — cobrem todo o fluxo de puxar deltas do backend e resolver conflitos.
- **Purchase order wizard:** cálculo de itens, aplicação de descontos por volume, comissão hierárquica RTV.
- **Auth:** login, refresh token, expiração, sync `authNotifier` ↔ `authTokenHolder`.
- **Repositórios Hive:** `BaseRepository.getList` com filtros por campo, `save/delete`.

## Regressões conhecidas historicamente

Testes escritos após incidentes:

- `sync_flow_integration_test.dart` — cenários de falha do sync (após incidente de sync silencioso quebrando).
- `client_address_map_test.dart` — não existe hoje, mas convém adicionar após a migração pra `flutter_map`.
- `refresh_token_test.dart` — após bug de `TimeSpan → double` no AutoMapper que travava mobile no spinner.

## Troubleshooting

**`Actual: _WidgetPredicateWidgetFinder:<Found 0 widgets…>`** — geralmente a versão do Flutter no CI é mais nova que a local, e um nome de classe privado mudou. Trocar por `find.byType(...)` ou `find.byWidgetPredicate((w) => w is …)`.

**`Some tests failed. Failing tests: ` no CI mas passa local** — SDK drift. O buildspec agora trava `FLUTTER_VERSION=3.38.9`; se ainda assim divergir, `flutter --version` no seu terminal deve bater com o CI.

**`Exception: Products failed` durante teste de sync** — comportamento intencional: `sync_flow_integration_test.dart > Error Handling` valida que o sync propaga a exceção corretamente.
