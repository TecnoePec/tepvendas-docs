# Mobile (Flutter 3.38)

App **Flutter/Dart** para Android + iOS. Segue a arquitetura offline-first: toda tela lê do **Hive** local; a sincronização com a API acontece em background via **audit log** (delta) + full-load na primeira execução.

## Stack

| Camada | Escolha |
|---|---|
| Framework | Flutter 3.38.9 (versão travada no CI) |
| State management | Riverpod (`StateNotifier` + `Provider`) |
| DB local | Hive (`hive`/`hive_flutter`) |
| HTTP | `dio` (chamadas de API) + `http` (integrações leves) |
| Formulários | `dropdown_search`, `mask_text_input_formatter`, `flutter_slidable` |
| Mapa | **`flutter_map`** (OSM) + **OSRM** público (rotas) |
| Localização | `geolocator`, `geocoding` |
| Auth | `jwt_decoder` + `encrypt` |
| Analytics | Firebase Analytics + Crashlytics |
| Config remota | Firebase Remote Config (com fallback `.env` via `EnvConfig`) |

## Estrutura de diretórios

```
lib/
├── app/
│   ├── analytics/           # AppAnalytics (Crashlytics wrapper)
│   ├── components/          # Widgets reutilizáveis
│   ├── data/
│   │   └── local_storage/   # Hive database (repositórios genéricos)
│   ├── domain/
│   │   ├── entities/        # Entities Hive (@HiveType)
│   │   ├── dtos/            # DTOs de resposta
│   │   └── interfaces/      # Abstrações
│   ├── enums/               # Enums de negócio
│   ├── guards/              # Route guards (auth)
│   ├── interfaces/          # Contratos de service/repo
│   ├── providers/           # Riverpod providers
│   ├── repositories/        # Wrappers em Hive + API
│   ├── services/
│   │   ├── auth/            # Login, refresh token
│   │   └── sync/            # Sync engine (audit-driven)
│   └── widgets/             # Páginas e sub-widgets
├── remote_config/           # CustomRemoteConfig + EnvConfig
├── theme/                   # Tema, cores, tipografia
└── utils/                   # Globals, helpers, formatters
```

## Sincronização (audit-driven)

O backend registra toda mudança em `audit_logs` via `AuditSaveChangesInterceptor`. O mobile pergunta periodicamente por audits mais novos que o último `lastSyncAt` e aplica os deltas nas boxes Hive:

```
[Mobile]                               [API]
   |  GET /audits?startDate=…&…        |
   +---------------------------------->|
   |  200 [{Add|Update|Delete rows}]   |
   |<----------------------------------+
   |
   | for each audit:
   |   → Hive.put(entityName, id, data)  (Add/Update)
   |   → Hive.delete(entityName, id)     (Delete)
```

Detalhes: [Segurança/Auditoria](../seguranca/auditoria.md).

## Mapa (endereço do cliente)

A tela `ClientAddressMap` usa **`flutter_map`** com tiles do OpenStreetMap e rotas via **OSRM público** (`router.project-osrm.org`, sem auth).

**Por que OSM em vez de Google Maps:**

- ISO 27001 A.14 (secure development) / A.15 (supplier relationships): elimina uma chave da Google Maps Platform (menos um segredo pra rotacionar, menos superfície de ataque).
- Remove risco de billing-DoS por quota estourada.
- Reduz a 1 as libs de mapa no app (o resto do app já usava `flutter_map`; havia `google_maps_flutter` só nessa tela).

**Comportamento preservado:**

- Marcadores de origem + centro de distribuição.
- Polyline de rota (OSRM); fallback pra distância haversine (`Geolocator.distanceBetween`) se OSRM falhar/atingir timeout.
- Geocoding de texto→coord via `geocoding` (independente do SDK Google Maps).
- Fit bounds pra englobar origem + destino.

## Convenções

- **Idioma código:** inglês
- **Idioma UI/mensagens:** pt-BR
- **Null safety:** ativa; evitar `!` em favor de `?.` + fallback (padrão adotado após crashes recorrentes no wizard).
- **Providers Riverpod:** sempre nullable no consumidor; guard cedo (early-return `SizedBox.shrink()`) se o dado obrigatório não chegou ainda.

## Testes

- **Unit + widget:** `flutter test` (~530+ testes).
- **Cobertura:** publicada via `flutter test --coverage`.
- **CI:** `flutter analyze --no-fatal-infos --no-fatal-warnings` + `flutter test` no `buildspec.yml`.

Detalhes: [Testes/Mobile](../testes/mobile.md).
