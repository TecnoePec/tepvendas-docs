# Gestão de Segredos

Normas de manipulação de credenciais, tokens e chaves no TEP Vendas — alinhadas à **ISO 27001 A.8.24 (uso de criptografia)** e **A.9.4.3 (gestão de senhas)**.

## Princípio central

> **Nenhum segredo em texto claro no repositório, em log ou em resposta HTTP.**

Todo segredo vive em **AWS Secrets Manager** (backend / infra) ou **Firebase Remote Config** (mobile), lido em runtime.

## Inventário de segredos

### Backend (`tep_vendas_services`)

| Secret | Consumidor | Rotação |
|---|---|---|
| `development.TEPVENDAS_CONNECTION_STRING` | ECS task var `POSTGRESQL_CONNECTION` | Aurora `ManageMasterUserPassword` — sync via Lambda |
| `development.TEPVENDAS_JWT_KEY` | ECS task var `Jwt__SecretKey` | Manual (rara) |
| `development.TEPVENDAS_REDIS_URL` | ECS task var `Redis__ConnectionString` | Manual |
| `development.TEPVENDAS_FIREBASE_SA` | CodeBuild env (Firebase App Distribution) | Manual (regenera SA key no Firebase Console) |

### Mobile (`tep_vendas_mobile`)

Firebase Remote Config keys (com fallback `.env` via `EnvConfig`):

- `base_url_api` (default: `https://tepvendas-api.tecnoepec.com.br`)
- `encryption_key` (Hive at-rest encryption)
- `sync_periodicity`
- `maintenance_mode` / `min_app_version` / `force_update`

!!! danger "Removido — não voltar"
    A chave `google_api_key` (Google Maps SDK) foi **removida** em ago/2026 junto com a migração de `google_maps_flutter` → `flutter_map` + OSRM. Não reintroduzir chaves Google Maps sem discussão de trade-offs (billing, secret rotation, dep externa).

## Regras práticas

### 1. Nunca commitar segredos em fallback

**Errado** (aconteceu em 2023, remediado em 2026):
```dart
RemoteConfigKeys.googleApiKey: EnvConfig().get('GOOGLE_API_KEY',
    defaultValue: '<REDACTED-google-maps-key>'),
//                ^^^^^^^^^^^^ chave real hard-coded como default
```

**Certo** — se a chave é obrigatória, deixa vazio ou levanta:
```dart
RemoteConfigKeys.encryptionKey: EnvConfig().get('ENCRYPTION_KEY',
    defaultValue: ''),
// Se vier vazio no runtime, o app loga erro visível e refuso o boot.
```

### 2. Placeholder em plist/manifest é lixo

`YOUR_IOS_GOOGLE_MAPS_API_KEY` no `Info.plist` era placeholder inerte. Confunde reviewer, aumenta ruído em audit. Se a config não é usada, **remove** o bloco inteiro.

### 3. Credencial exposta = credencial comprometida

Se um segredo apareceu em git, log, PR review ou screenshot: **revoga imediatamente** — mesmo que "só o time viu". A chave continua no `git log --all -S 'AIza…'` até refazer o history.

!!! warning "Não escreve a chave literal em doc de post-mortem"
    O scanner do Google Cloud Trust & Safety detecta o formato `AIza…` em qualquer URL pública que ele indexar — incluindo esta documentação. Um doc de incidente que **cita** a chave literal, mesmo pra registrar o que foi deletado, dispara alerta como se o vazamento fosse novo. **Sempre mascara** (ex: `<REDACTED-google-maps-key>`, ou prefixo curto `AIzaSyBw2…`) e mantém a identificação real (project ID, key UID, data) no incidente.

**Fluxo pós-vazamento** (ISO 27001 A.16.1 — resposta a incidente):

1. Remove do código (commit + push)
2. Revoga no console do provedor (AWS Secrets Manager `delete-secret`, Google Cloud `gcloud services api-keys delete`, Firebase Console → rotate)
3. Documenta no `docs/seguranca/gestao-segredos.md` (essa página) na tabela "Incidentes históricos"
4. Se afetou produção: notifica stakeholders

### 4. Rotação de secret ≠ update do consumidor

Aurora rotaciona senha via `ManageMasterUserPassword`, mas o secret `development.TEPVENDAS_CONNECTION_STRING` **não** é atualizado automaticamente. Uma Lambda faz o sync — ver [project_aurora_sync_lambda](https://github.com/TecnoePec/tepvendas-infra) na infra.

Depois da rotação, ECS task precisa reiniciar pra pegar o novo secret (mount de secret é on-boot, não on-change).

## CI/CD

O CodeBuild não deve **imprimir** valores de secret em log. Verifica no `buildspec.yml`:

- **OK:** `secrets-manager: { FOO: 'name-of-secret' }` — CodeBuild masks value in logs
- **Não OK:** `echo $FOO` — expõe se debug flag estiver ligado

## Incidentes históricos

| Data | Segredo | Detecção | Remediação |
|---|---|---|---|
| ago/2026 | `<REDACTED-google-maps-key>` (Google Maps API Key, projeto GCP `pc-api-8870686889883584396-428`) | Encontrado em `custom_remote_config.dart:47` durante refactor do mapa | (a) Removido do código, (b) `google_maps_flutter` substituído por `flutter_map`, (c) chave **deletada** via `gcloud services api-keys delete` |

## Ferramentas recomendadas

- **[gitleaks](https://github.com/gitleaks/gitleaks)** — scan do repo antes de push (`gitleaks detect --source .`)
- **AWS Secrets Manager rotation** — configurar rotation automática pra RDS quando possível
- **Firebase Remote Config Personalization** — usar em vez de hard-code de defaults
