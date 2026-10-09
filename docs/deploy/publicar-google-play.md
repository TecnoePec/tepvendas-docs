# Publicar AAB no Google Play — runbook

A pipeline mobile já gera `app-release.aab` em todo build. Pra publicar
automaticamente no Google Play Console (track Internal / Alpha / Beta /
Production), falta apenas o Service Account configurado + o flag na pipeline.

## Setup único (fazer agora)

### 1. Provisionar a Service Account via Terraform

Toda a parte GCP + AWS Secrets Manager é um módulo TF auto-contido em
[tep_vendas_mobile/infra/tf/google_play_sa/](https://github.com/TecnoePec/tep_vendas_mobile/tree/develop/infra/tf/google_play_sa)
— o README do módulo tem o passo-a-passo completo. Resumo:

```bash
gcloud auth application-default login
aws sso login --profile tecnoepec-dev

cd tep_vendas_mobile/infra/tf/google_play_sa
terraform init
terraform apply
```

Isso cria:

- Service Account `play-publisher@tepvenda.iam.gserviceaccount.com`
- Habilita APIs `iamcredentials` + `androidpublisher` no project `tepvenda`
- Chave privada RSA 2048 (JSON)
- Secret `development.TEPVENDAS_PLAY_SA` no AWS Secrets Manager com o JSON

Rotacionar depois: `terraform taint google_service_account_key.play_publisher && terraform apply`.

### 2. Convidar a SA no Google Play Console (manual — API não existe)

Terraform não cobre essa etapa porque o Play Console tem IAM próprio
separado do Google Cloud IAM.

1. https://play.google.com/console → **Users and permissions** → **Invite new users**
2. Email: valor do output `service_account_email` do TF (`play-publisher@tepvenda.iam.gserviceaccount.com`)
3. Permissions → **Admin (all permissions)** OU específicas: Release manager + View app information
4. Apps access → seleciona TEP Vendas
5. Invite — SA aparece ativa imediatamente

### 3. Upload manual do primeiro AAB pro Play Console

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
