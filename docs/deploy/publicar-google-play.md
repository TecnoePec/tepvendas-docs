# Publicar AAB no Google Play — runbook

A pipeline mobile já gera `app-release.aab` em todo build. Pra publicar
automaticamente no Google Play Console (track Internal / Alpha / Beta /
Production), falta apenas o Service Account configurado + o flag na pipeline.

## Setup único (fazer agora)

### 1. Criar Service Account no Google Play Console

1. Abre https://play.google.com/console → Setup → **API access**
2. Se for a primeira vez: Link Google Cloud project (associa com o project
   `tepvenda` que já existe)
3. Em **Service accounts** → **Create new service account** → te leva pro
   Google Cloud IAM
4. Nome: `play-publisher`, Role: nenhum (perms vêm do Play Console depois)
5. Volta pro Play Console → recarrega → o SA aparece na lista →
   **Grant access** → marque:
   - **Admin (all permissions)** OU
   - Permissions específicas: Release manager + View app information
6. Em cima do SA → **View details** → **Keys** tab → **Add key** → JSON →
   baixa o arquivo (não compartilhar).

### 2. Subir pro AWS Secrets Manager

```bash
aws --profile tecnoepec-dev secretsmanager create-secret \
  --name development.TEPVENDAS_PLAY_SA \
  --description "Service Account JSON do Google Play Console para publicação automática do AAB" \
  --secret-string file://~/Downloads/pc-api-XXXX-YYY.json \
  --region us-east-1
```

Depois apaga o download local:
```bash
rm ~/Downloads/pc-api-*.json
```

### 3. Fazer upload manual do primeiro AAB pro Play Console

O Google Play só aceita upload via API **depois** que você já publicou pelo
menos uma versão manualmente pela UI. Baixa o `app-release.aab` do último
pipeline (está em S3 artifacts) ou roda `flutter build appbundle --release`
local e sobe uma vez em **Internal Testing**.

Depois disso o SA pode publicar via API nas releases seguintes.

## Como publicar pela pipeline

Dispara a pipeline mobile pela UI do CodePipeline passando env override:

```
PUBLISH_PLAY=true
PLAY_TRACK=internal       # ou alpha, beta, production
```

O buildspec, no `post_build`, vai:
1. Instalar fastlane
2. Buscar `development.TEPVENDAS_PLAY_SA` do Secrets Manager
3. Rodar `fastlane publish_play`:
   - `fastlane-plugin-upload_to_play_store` upa o AAB pro track especificado
   - Pula metadata/images/screenshots/changelog (ficam gerenciados manual na UI)
4. Limpar o SA local do runner

Default `PLAY_TRACK=internal` — não publica pra usuário final, só pra lista
"Internal testers" cadastrada no Play Console.

## Promover de Internal → Produção

Pelo próprio Play Console: Releases → **Internal testing** → selecionar a
release → **Promote to production**. Dá pra fazer canary (ex: 5% dos
usuários) na mesma UI.

Alternativa automatizada: rodar a pipeline de novo com
`PLAY_TRACK=production` — mas só se o fluxo de review manual já tiver sido
feito uma vez.

## Troubleshooting

**"The caller does not have permission"** — SA não tem a role certa no Play
Console. Garante que marcou "Release manager" ou "Admin" lá.

**"Version code X already used"** — bump o `version:` no `pubspec.yaml`
(formato `1.0.0+N` — o N é o versionCode Android, cada upload precisa ser
único).

**"Changes may not be sent to review"** — adiciona
`PLAY_CHANGES_NOT_REVIEWED=true` à override. Útil pra releases de teste.
