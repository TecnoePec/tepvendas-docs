# CI/CD Setup - TEP Vendas

**Sprint 3.5 - Continuous Integration & Continuous Deployment**
**Data:** 30 Jan 2026
**Status:** ✅ Implementado

---

## Índice

1. [Visão Geral](#visão-geral)
2. [Backend API (.NET)](#backend-api-net)
3. [Web Frontend (Next.js)](#web-frontend-nextjs)
4. [Mobile App (Flutter)](#mobile-app-flutter)
5. [Configuração de Secrets](#configuração-de-secrets)
6. [Workflows Locais](#workflows-locais)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Roadmap](#roadmap)

---

## Visão Geral

Este projeto implementa pipelines completos de CI/CD usando **GitHub Actions** para automatizar:
- ✅ Build
- ✅ Testes automatizados
- ✅ Análise de código e lint
- ✅ Cobertura de código
- ✅ Scan de segurança
- ✅ Quality gates
- ✅ Deploy automatizado (configurável)

### Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Repository                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Push/PR to develop/main                                    │
│           ↓                                                  │
│  ┌───────────────────────────────────────────────┐         │
│  │         GitHub Actions Workflows              │         │
│  ├───────────────────────────────────────────────┤         │
│  │                                               │         │
│  │  Backend (.NET)    Web (Next.js)    Mobile (Flutter) │  │
│  │       ↓                 ↓                 ↓           │  │
│  │    Build            Build             Build          │  │
│  │    Test             Test              Test           │  │
│  │    Coverage         Coverage          Coverage       │  │
│  │    Security         Lint              Analyze        │  │
│  │    Quality Gate     Quality Gate      Quality Gate   │  │
│  │       ↓                 ↓                 ↓           │  │
│  │    Docker           Artifacts         APK/IPA        │  │
│  │       ↓                 ↓                 ↓           │  │
│  │    Deploy           Deploy            Deploy         │  │
│  │   (Optional)       (Optional)         (Optional)     │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Plataformas

| Plataforma | Framework | Workflow File | Status |
|------------|-----------|---------------|--------|
| **Backend API** | .NET 6.0 | [`dotnet-core-build.yml`](tep_vendas_services/.github/workflows/dotnet-core-build.yml) | ✅ Ativo |
| **Web Frontend** | Next.js 12 | [`nextjs-build-test.yml`](tep_web_app/.github/workflows/nextjs-build-test.yml) | ✅ Ativo |
| **Mobile App** | Flutter 3.24+ | [`flutter-build-test.yml`](TEP_Vendas_Mobile/.github/workflows/flutter-build-test.yml) | ✅ Ativo |

---

## Backend API (.NET)

### Arquivo de Workflow

**Localização:** `tep_vendas_services/.github/workflows/dotnet-core-build.yml`

### Triggers

```yaml
on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]
  workflow_dispatch: # Manual trigger via GitHub UI
```

### Jobs

#### 1. Build & Test (build-and-test)

**Duração:** ~5-7 minutos

**Steps:**
1. ✅ Checkout do código
2. ✅ Setup .NET 6.0
3. ✅ Cache de pacotes NuGet
4. ✅ Restore de dependências
5. ✅ Build em Release
6. ✅ Run testes com cobertura
7. ✅ Upload de artifacts (test results + coverage)
8. ✅ Geração de relatório de cobertura
9. ✅ Comentário de coverage em PRs

**Comandos principais:**
```bash
dotnet restore src/Tep.Sales.Service.sln
dotnet build src/Tep.Sales.Service.sln --configuration Release --no-restore
dotnet test src/Tep.Sales.Service.sln \
  --configuration Release \
  --no-build \
  --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover
```

**Artifacts gerados:**
- `test-results.trx` - Resultados dos testes
- `coverage.opencover.xml` - Relatório de cobertura

#### 2. Security Scan (security-scan)

**Duração:** ~2-3 minutos

**Verifica:**
- ✅ Pacotes vulneráveis
- ✅ Dependências transitivas
- ✅ Pacotes desatualizados

```bash
dotnet list package --vulnerable --include-transitive
dotnet list package --outdated
```

#### 3. Quality Gate (quality-gate)

**Duração:** ~1 minuto

**Valida:**
- ✅ Cobertura de código foi gerada
- ℹ️ Futuro: enforçar thresholds mínimos

#### 4. Build Docker (build-docker)

**Duração:** ~3-5 minutos
**Condição:** Apenas em `main` ou `develop`

**Build:**
- Imagem Docker para deployment
- Cache otimizado com GitHub Actions cache

```dockerfile
# Exemplo de Dockerfile (não incluído no projeto ainda)
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
# ... build steps
```

#### 5. Deployment Notification (deployment-notification)

**Condição:** Sucesso em `main` ou `develop`

Notifica que a aplicação está pronta para deploy.

#### 6. Summary (summary)

Gera resumo visual no GitHub Actions.

### Configuração Local

**Pré-requisitos:**
```bash
# Instalar .NET 6 SDK
# macOS
brew install --cask dotnet-sdk6

# Ubuntu
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 6.0

# Verificar instalação
dotnet --version  # Deve ser 6.0.x
```

**Rodar localmente:**
```bash
cd tep_vendas_services

# Restore
dotnet restore src/

# Build
dotnet build src/ --configuration Release

# Test
dotnet test src/ --configuration Release

# Test com coverage
dotnet test src/ --configuration Release \
  --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

# Ver relatório de coverage
# Instalar ReportGenerator
dotnet tool install -g dotnet-reportgenerator-globaltool

# Gerar relatório HTML
reportgenerator \
  -reports:"**/coverage.opencover.xml" \
  -targetdir:"coveragereport" \
  -reporttypes:Html

# Abrir relatório
open coveragereport/index.html  # macOS
```

### Variáveis de Ambiente

**Configurar no GitHub:**
1. Settings → Secrets and variables → Actions
2. Adicionar secrets:

| Secret | Descrição | Exemplo |
|--------|-----------|---------|
| `SENTRY_DSN` | Sentry DSN para backend | `https://...@o000.ingest.sentry.io/...` |
| `DOCKER_USERNAME` | Username do registry Docker | `seu-usuario` |
| `DOCKER_PASSWORD` | Password do registry Docker | `seu-token` |

---

## Web Frontend (Next.js)

### Arquivo de Workflow

**Localização:** `tep_web_app/.github/workflows/nextjs-build-test.yml`

### Triggers

Mesmos triggers do backend (push/PR em develop/main + manual).

### Jobs

#### 1. Lint & Code Quality (lint)

**Duração:** ~1-2 minutos

**Verifica:**
- ✅ ESLint
- ✅ Prettier formatting

```bash
npm ci
npm run lint
npx prettier --check "src/**/*.{ts,tsx,js,jsx,json,css,scss,md}"
```

#### 2. Test & Coverage (test)

**Duração:** ~3-5 minutos

**Executa:**
- ✅ Jest tests
- ✅ Cobertura de código
- ✅ Geração de relatórios

```bash
npm ci
npm run test:coverage
```

**Artifacts gerados:**
- `coverage/` - Relatório completo
- `coverage/lcov.info` - Para ferramentas externas
- `coverage/coverage-summary.json` - Resumo JSON

#### 3. Build Development (build-dev)

**Duração:** ~3-5 minutos

**Build para ambiente de desenvolvimento:**
```bash
npm ci
npm run build
```

**Env vars:**
- `NEXT_PUBLIC_API_BASE_URL=https://dev-api.tecnoepec.com.br`
- `NEXT_PUBLIC_SENTRY_DSN` (do secret)
- `NEXT_PUBLIC_SENTRY_ENVIRONMENT=development`

**Artifacts:**
- `.next/` build folder
- `public/` static assets

#### 4. Build Production (build-prod)

**Duração:** ~3-5 minutos
**Condição:** Apenas em `main`

**Build para produção:**
```bash
npm ci
npm run build
```

**Env vars:**
- `NEXT_PUBLIC_API_BASE_URL=https://prd-api.tecnoepec.com.br`
- `NEXT_PUBLIC_SENTRY_DSN` (do secret)
- `NEXT_PUBLIC_SENTRY_ENVIRONMENT=production`
- `SENTRY_AUTH_TOKEN` (para source maps)

#### 5. Security Audit (security)

**Duração:** ~1-2 minutos

```bash
npm audit --audit-level=moderate
```

Gera relatório de vulnerabilidades no summary.

#### 6. Quality Gate (quality-gate)

**Valida:**
- ✅ Lint passou
- ✅ Tests passaram
- ✅ Build passou

Falha se qualquer um desses jobs falhou.

#### 7. Deploy Preview (deploy-preview)

**Condição:** `develop` ou PRs

Prepara deployment preview (Vercel, Netlify, etc.).

#### 8. Deploy Production (deploy-production)

**Condição:** Apenas `main`
**Environment:** `production`

URL: https://vendas.tecnoepec.com.br

#### 9. Summary (summary)

Resumo consolidado do pipeline.

### Configuração Local

**Pré-requisitos:**
```bash
# Instalar Node.js 18+
# macOS
brew install node@18

# Ubuntu
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verificar
node --version  # v18.x.x
npm --version   # 9.x.x
```

**Rodar localmente:**
```bash
cd tep_web_app

# Instalar dependências
npm install

# Lint
npm run lint

# Tests
npm run test
npm run test:watch        # Watch mode
npm run test:coverage     # Com coverage

# Build development
npm run build-development

# Build production
npm run build-production

# Rodar dev server
npm run dev

# Ver coverage report
open coverage/lcov-report/index.html  # macOS
```

### Variáveis de Ambiente

**GitHub Secrets:**

| Secret | Descrição | Exemplo |
|--------|-----------|---------|
| `SENTRY_DSN_DEV` | Sentry DSN dev/staging | `https://...@o000.ingest.sentry.io/...` |
| `SENTRY_DSN_PROD` | Sentry DSN produção | `https://...@o000.ingest.sentry.io/...` |
| `SENTRY_AUTH_TOKEN` | Token para upload source maps | `sntrys_...` |
| `SENTRY_ORG` | Nome da org no Sentry | `tecnoepec` |
| `SENTRY_PROJECT` | Nome do projeto | `tep-vendas-web` |
| `VERCEL_TOKEN` | Token do Vercel (se usar) | `...` |
| `VERCEL_ORG_ID` | Org ID do Vercel | `...` |
| `VERCEL_PROJECT_ID` | Project ID do Vercel | `...` |

**Arquivo `.env.local` (local development):**
```bash
# API
NEXT_PUBLIC_API_BASE_URL=http://localhost:5000

# Sentry
NEXT_PUBLIC_SENTRY_DSN=https://...@o000.ingest.sentry.io/...
NEXT_PUBLIC_SENTRY_ENVIRONMENT=development
SENTRY_DSN=https://...@o000.ingest.sentry.io/...
SENTRY_ENVIRONMENT=development
```

---

## Mobile App (Flutter)

### Arquivo de Workflow

**Localização:** `TEP_Vendas_Mobile/.github/workflows/flutter-build-test.yml`

### Triggers

Mesmos triggers (push/PR em develop/main + manual).

### Jobs

#### 1. Analyze & Lint (analyze)

**Duração:** ~2-3 minutos

**Verifica:**
- ✅ Formatação do código Dart
- ✅ Análise estática (flutter analyze)
- ✅ Pacotes desatualizados

```bash
flutter pub get
dart format --output=none --set-exit-if-changed .
flutter analyze --no-fatal-infos --no-fatal-warnings
flutter pub outdated
```

#### 2. Test & Coverage (test)

**Duração:** ~3-5 minutos

**Executa:**
- ✅ Unit tests
- ✅ Widget tests
- ✅ Cobertura de código

```bash
flutter pub get
flutter test --coverage --reporter expanded
```

**Artifact:**
- `coverage/lcov.info`

#### 3. Integration Tests (integration-test)

**Duração:** Variável
**Status:** Desabilitado (configurar quando pronto)

```bash
flutter test integration_test/
```

#### 4. Build Android APK Development (build-android-dev)

**Duração:** ~5-8 minutos

**Build:**
- APK Debug
- Target: android-arm64

```bash
flutter pub get
flutter build apk --debug --target-platform android-arm64
```

**Artifact:**
- `app-debug.apk`

#### 5. Build Android Production (build-android-prod)

**Duração:** ~8-12 minutos
**Condição:** Apenas `main`

**Build:**
- App Bundle (`.aab`) para Play Store
- APKs splitados por ABI

```bash
flutter build appbundle --release
flutter build apk --release --split-per-abi
```

**Artifacts:**
- `app-release.aab`
- `app-armeabi-v7a-release.apk`
- `app-arm64-v8a-release.apk`
- `app-x86_64-release.apk`

**⚠️ Nota:** Para produção, é necessário configurar keystore para assinatura.

#### 6. Build iOS (build-ios)

**Duração:** ~10-15 minutos
**Runner:** `macos-latest`
**Condição:** `main` ou `develop`

```bash
flutter pub get
flutter build ios --release --no-codesign
```

**⚠️ Nota:** Para produção real, precisa de certificados e provisioning profiles da Apple.

#### 7. Security Scan (security)

**Duração:** ~1-2 minutos

```bash
flutter pub outdated
```

#### 8. Quality Gate (quality-gate)

**Valida:**
- ✅ Analyze passou
- ✅ Tests passaram
- ✅ Build Android passou

#### 9. Deploy to Firebase App Distribution (deploy-firebase)

**Condição:** `develop`

Distribui APK para beta testers via Firebase App Distribution.

**⚠️ Requer configuração:**
```yaml
# Adicionar ao workflow:
- uses: wzieba/Firebase-Distribution-Github-Action@v1
  with:
    appId: ${{ secrets.FIREBASE_APP_ID }}
    token: ${{ secrets.FIREBASE_TOKEN }}
    groups: beta-testers
    file: app-debug.apk
```

#### 10. Deploy to Play Store (deploy-playstore)

**Condição:** `main`
**Environment:** `production-android`

URL: https://play.google.com/store/apps/details?id=br.com.tecnoepec.vendas

**⚠️ Requer configuração:**
```yaml
# Adicionar ao workflow:
- uses: r0adkll/upload-google-play@v1
  with:
    serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
    packageName: br.com.tecnoepec.vendas
    releaseFiles: build/app/outputs/bundle/release/*.aab
    track: production
```

#### 11. Summary (summary)

Resumo do pipeline.

### Configuração Local

**Pré-requisitos:**
```bash
# Instalar Flutter
# macOS
brew install --cask flutter

# Ubuntu
sudo snap install flutter --classic

# Verificar instalação
flutter doctor

# Aceitar licenças Android
flutter doctor --android-licenses

# Instalar dependências
flutter pub get
```

**Rodar localmente:**
```bash
cd TEP_Vendas_Mobile

# Get dependencies
flutter pub get

# Análise estática
dart format .
flutter analyze

# Tests
flutter test
flutter test --coverage

# Ver coverage
# Instalar lcov (macOS)
brew install lcov

# Gerar HTML
genhtml coverage/lcov.info -o coverage/html

# Abrir
open coverage/html/index.html

# Build Android
flutter build apk --debug
flutter build apk --release

# Build iOS (macOS only)
flutter build ios --release --no-codesign

# Run app
flutter run
flutter run --release
```

### Variáveis de Ambiente

**GitHub Secrets:**

| Secret | Descrição | Exemplo |
|--------|-----------|---------|
| `SENTRY_DSN_DEV` | Sentry DSN dev | `https://...` |
| `SENTRY_DSN_PROD` | Sentry DSN prod | `https://...` |
| `FIREBASE_APP_ID` | Firebase App ID | `1:000000000000:android:...` |
| `FIREBASE_TOKEN` | Firebase CI token | `1//...` |
| `PLAY_STORE_SERVICE_ACCOUNT` | JSON da service account | `{...}` |
| `ANDROID_KEYSTORE_BASE64` | Keystore em base64 | `...` |
| `ANDROID_KEY_ALIAS` | Alias da chave | `tecnoepec` |
| `ANDROID_KEY_PASSWORD` | Senha da chave | `...` |
| `ANDROID_STORE_PASSWORD` | Senha do keystore | `...` |

**Arquivo `.env` (root do projeto mobile):**
```bash
# API
API_BASE_URL=http://10.0.2.2:5000  # Android emulator

# Sentry
SENTRY_DSN=https://...@o000.ingest.sentry.io/...

# Sync
SYNC_PERIODICITY=120
```

**Gerar keystore (para produção):**
```bash
keytool -genkey -v \
  -keystore android-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias tecnoepec

# Converter para base64 (para GitHub secret)
base64 android-keystore.jks | pbcopy  # macOS
base64 android-keystore.jks | xclip   # Linux
```

**Configurar keystore no Flutter:**

Criar `android/key.properties`:
```properties
storePassword=sua_senha
keyPassword=sua_senha
keyAlias=tecnoepec
storeFile=../android-keystore.jks
```

Adicionar ao `.gitignore`:
```
android/key.properties
android-keystore.jks
```

---

## Configuração de Secrets

### Como Adicionar Secrets no GitHub

1. Ir para **Settings** do repositório
2. **Secrets and variables** → **Actions**
3. **New repository secret**
4. Adicionar nome e valor
5. Salvar

### Secrets Necessários

#### Para todos os projetos:
- `SENTRY_DSN_DEV`
- `SENTRY_DSN_PROD`

#### Para Web (Next.js):
- `SENTRY_AUTH_TOKEN`
- `SENTRY_ORG`
- `SENTRY_PROJECT`
- `VERCEL_TOKEN` (se usar Vercel)
- `VERCEL_ORG_ID`
- `VERCEL_PROJECT_ID`

#### Para Mobile (Flutter):
- `FIREBASE_APP_ID`
- `FIREBASE_TOKEN`
- `PLAY_STORE_SERVICE_ACCOUNT`
- `ANDROID_KEYSTORE_BASE64`
- `ANDROID_KEY_ALIAS`
- `ANDROID_KEY_PASSWORD`
- `ANDROID_STORE_PASSWORD`

#### Para Backend (.NET):
- `DOCKER_USERNAME` (se usar Docker Hub)
- `DOCKER_PASSWORD`
- `AZURE_CREDENTIALS` (se usar Azure)

### Obter Tokens

**Sentry Auth Token:**
```bash
# 1. Login no Sentry: https://sentry.io/
# 2. Settings → Account → API → Auth Tokens
# 3. Create New Token
# 4. Scopes: project:releases, project:write
# 5. Copiar token
```

**Firebase Token:**
```bash
# Instalar Firebase CLI
npm install -g firebase-tools

# Login
firebase login:ci

# Copiar token gerado
```

**Play Store Service Account:**
```bash
# 1. Google Cloud Console
# 2. IAM & Admin → Service Accounts
# 3. Create Service Account
# 4. Grant Play Developer role
# 5. Create JSON key
# 6. Copiar conteúdo do JSON
```

---

## Workflows Locais

### Act - Run GitHub Actions Locally

**Instalação:**
```bash
# macOS
brew install act

# Ubuntu
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
```

**Uso:**
```bash
# Listar workflows
act -l

# Rodar workflow específico
act push  # Simula push
act pull_request  # Simula PR

# Rodar job específico
act -j build-and-test

# Com secrets
act push --secret-file .secrets
```

**Arquivo `.secrets`:**
```bash
SENTRY_DSN=your_dsn_here
DOCKER_PASSWORD=your_password_here
```

### Pre-commit Hooks

**Instalar:**
```bash
# Backend (.NET)
cd tep_vendas_services
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
dotnet build src/ --no-restore
dotnet test src/ --no-build
EOF
chmod +x .git/hooks/pre-commit

# Web (Next.js)
cd tep_web_app
npx husky install
npx husky add .husky/pre-commit "npm run lint && npm test"

# Mobile (Flutter)
cd TEP_Vendas_Mobile
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
dart format .
flutter analyze
flutter test
EOF
chmod +x .git/hooks/pre-commit
```

---

## Best Practices

### 1. Branch Strategy

**Fluxo recomendado:**
```
feature/xxx → develop → main
    ↓            ↓        ↓
  CI only    CI + Preview  CI + Production
```

**Branches:**
- `main` - Produção (protegida)
- `develop` - Staging/Development (protegida)
- `feature/*` - Features (CI em PRs)
- `hotfix/*` - Correções urgentes

### 2. Pull Requests

**Antes de criar PR:**
```bash
# Rodar localmente
npm run lint      # ou flutter analyze / dotnet build
npm test          # ou flutter test / dotnet test
npm run build     # ou flutter build / dotnet build

# Verificar coverage
npm run test:coverage

# Commit apenas se tudo passar
```

**Template de PR:**
```markdown
## Description
[Descrição da mudança]

## Type of change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Checklist
- [ ] Tests pass locally
- [ ] Lint passes
- [ ] Coverage não diminuiu
- [ ] Documentation updated
```

### 3. Commits

**Conventional Commits:**
```bash
feat: add new login form
fix: correct validation in payment
test: add tests for user service
docs: update CI/CD documentation
chore: update dependencies
```

### 4. Debugging Workflows

**Ver logs:**
1. GitHub → Actions tab
2. Clicar no workflow run
3. Clicar no job
4. Ver logs detalhados

**Re-run failed jobs:**
1. Clicar em "Re-run failed jobs"
2. Ou "Re-run all jobs" para todos

**Adicionar debug logs:**
```yaml
- name: Debug
  run: |
    echo "Environment: ${{ github.ref }}"
    echo "Actor: ${{ github.actor }}"
    ls -la
```

### 5. Otimização de Performance

**Caching:**
- ✅ Sempre usar cache para dependências
- ✅ Cache de build (quando possível)
- ✅ Artifacts para compartilhar entre jobs

**Paralelização:**
- ✅ Jobs independentes rodam em paralelo
- ✅ Matrix builds para múltiplas versões

**Eficiência:**
- ✅ Fail fast quando possível
- ✅ Conditional jobs (`if:`)
- ✅ Cancelar runs anteriores em novo push

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## Troubleshooting

### Problema: Build passa localmente mas falha no CI

**Causas comuns:**
1. **Dependências não commitadas**
   - Verificar `.gitignore`
   - Rodar `npm ci` (não `npm install`)

2. **Diferenças de ambiente**
   - Versões diferentes de Node/Flutter/.NET
   - Variáveis de ambiente faltando

3. **Cache corrompido**
   ```yaml
   # Adicionar ao workflow para limpar cache
   - name: Clear cache
     run: rm -rf ~/.npm ~/.nuget ~/.pub-cache
   ```

**Solução:**
```bash
# Simular CI localmente
rm -rf node_modules/
npm ci
npm test && npm run build
```

### Problema: Testes flaky (passam às vezes, falham outras)

**Causas:**
1. Testes dependentes de timing
2. Testes dependentes de ordem
3. Testes dependentes de estado global

**Solução:**
```bash
# Rodar testes múltiplas vezes
npm test -- --runInBand  # Sequencial
npm test -- --maxWorkers=1
```

### Problema: Coverage não está sendo gerada

**Backend:**
```bash
# Verificar se Coverlet está instalado
dotnet add package coverlet.collector

# Testar localmente
dotnet test --collect:"XPlat Code Coverage"
```

**Web:**
```bash
# Verificar jest.config.js
cat jest.config.js | grep collectCoverageFrom

# Rodar localmente
npm run test:coverage
ls coverage/  # Verificar se gerou
```

**Mobile:**
```bash
# Rodar com coverage
flutter test --coverage

# Verificar arquivo
ls coverage/lcov.info
```

### Problema: Job demora muito (timeout)

**Solução:**
```yaml
jobs:
  my-job:
    timeout-minutes: 30  # Padrão é 360 (6h)
```

**Otimizações:**
- Usar cache
- Paralelizar jobs
- Build incremental

### Problema: Secrets não estão disponíveis

**Verificar:**
1. Secret está configurado no GitHub
2. Nome do secret está correto (case-sensitive)
3. Secret está disponível para o branch

**Uso correto:**
```yaml
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}
```

### Problema: Workflow não está rodando

**Verificar:**
1. Arquivo está em `.github/workflows/`
2. Extensão é `.yml` ou `.yaml`
3. Sintaxe YAML está correta
4. Trigger está configurado para o branch

**Validar YAML:**
```bash
# Online
https://www.yamllint.com/

# CLI
yamllint .github/workflows/*.yml
```

---

## Roadmap

### Sprint 3.5 (Atual) ✅
- [x] Workflows configurados para Backend, Web, Mobile
- [x] Testes automatizados
- [x] Cobertura de código
- [x] Quality gates informativos
- [x] Documentação completa

### Sprint 4.1 (Próximo)
- [ ] Deploy automático para staging (develop → staging)
- [ ] Deploy manual para produção (main → production)
- [ ] Notificações no Slack/Discord
- [ ] Badges de status no README

### Sprint 4.2
- [ ] SonarQube/SonarCloud integration
- [ ] Dependency updates automatizados (Dependabot/Renovate)
- [ ] Performance monitoring
- [ ] E2E tests no CI

### Sprint 4.3
- [ ] Quality gates restritivos (bloqueiam PRs)
- [ ] Deploy blue-green ou canary
- [ ] Rollback automático em caso de erro
- [ ] Testes de carga automatizados

### Sprint 5.x
- [ ] Multi-region deployment
- [ ] A/B testing infrastructure
- [ ] Chaos engineering
- [ ] GitOps com ArgoCD/Flux

---

## Recursos Adicionais

### Documentação Relacionada

- **Quality Gates:** [`QUALITY_GATES.md`](QUALITY_GATES.md)
- **Observability:** [`OBSERVABILITY_SETUP_README.md`](OBSERVABILITY_SETUP_README.md)
- **Roadmap:** [`ROADMAP_TEP_VENDAS.md`](ROADMAP_TEP_VENDAS.md)
- **Kanban:** [`KANBAN_TEP_VENDAS.md`](KANBAN_TEP_VENDAS.md)

### Links Úteis

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [.NET Testing Best Practices](https://learn.microsoft.com/en-us/dotnet/core/testing/)
- [Jest Testing](https://jestjs.io/docs/getting-started)
- [Flutter Testing](https://docs.flutter.dev/testing)
- [Act - Run GitHub Actions Locally](https://github.com/nektos/act)

### Contato

**Sprint:** 3.5 - CI/CD
**Data:** 30 Jan 2026
**Duração:** ~40h

Para dúvidas ou sugestões sobre CI/CD, abra uma issue no repositório.
