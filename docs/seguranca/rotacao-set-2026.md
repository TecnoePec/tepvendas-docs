# Runbook — Rotação pós-vazamento set/2026

Descoberto durante o setup do gitleaks: 4 segredos vivos estavam commitados
no git. Todos foram removidos do working tree e cobertos pelo scanner, mas
as credenciais **ainda são válidas** — quem baixou o repo antes da limpeza
tem acesso. Este runbook é a rotação.

Ordem sugerida: **do risco mais alto pro mais baixo.**

---

## 1. Firebase FCM Service Account privkey — CRÍTICO

**O que expõe:** capacidade de emitir tokens em nome do projeto Firebase
`tepvenda` (envio de push, acesso a Firestore/Storage se rules confiarem em
tokens do server).

**Rotate:**

1. Firebase Console → https://console.firebase.google.com/project/tepvenda/settings/serviceaccounts/adminsdk
2. Botão **"Generate new private key"** → download do JSON
3. **Desabilite** a chave antiga na mesma página (o botão fica ao lado de cada
   `key-id` na lista de "Chaves atuais"). Isso é o que efetivamente revoga
   quem já vazou.
4. Codifique o JSON novo em base64:
   ```bash
   base64 < caminho/para/tepvenda-firebase-adminsdk-NEW.json | pbcopy
   ```
5. Atualize o secret AWS:
   ```bash
   aws --profile tecnoepec-dev secretsmanager put-secret-value \
     --secret-id development.TEPVENDAS_FIREBASE_SA \
     --secret-string "$(cat tepvenda-firebase-adminsdk-NEW.json)" \
     --region us-east-1
   ```
6. Force nova task no ECS pra pegar o valor novo:
   ```bash
   aws --profile tecnoepec-dev ecs update-service --cluster main \
     --service tepvendas-api --force-new-deployment --region us-east-1
   ```
7. Apague o download local: `rm ~/Downloads/tepvenda-firebase-adminsdk-*.json`

---

## 2. Firebase Web API key `AIzaSyDZ3Vj…` — ALTO

**O que expõe:** ela é design-time "pública" (aparece no `google-services.json`
do app), mas serve pra autenticar anônimo no Firebase Storage do projeto — se
não estiver restringida por App bundle, qualquer um pode assinar.

**Opção A — Restringir (recomendado):**

1. https://console.cloud.google.com/apis/credentials?project=tepvenda
2. Localize "API key (auto created by Firebase)"
3. Application restrictions → **Android apps** → adicione `br.com.tecnoepec.tepvendas` com o SHA-1 da upload key (já em nossa doc, `keystore-backup/README.txt`)
4. API restrictions → só **Identity Toolkit API** + **Firebase Realtime Database API** + **Cloud Storage for Firebase API**

**Opção B — Rotate:**

1. Mesma tela, botão "Regenerate key"
2. Atualize no `google-services.json` do app (mobile) e no env `FIREBASE_WEB_API_KEY` da task ECS

Depois de restringir/rotate, teste subir uma imagem no backoffice — se
Storage retornar 403, a restrição está errada; abra pra ver o erro no console
do Firebase.

---

## 3. SAP B1 password `1234` — CRÍTICO

**O que expõe:** acesso admin ao SAP B1 da Major Nutrição em `b1.ativy.com:50211` com user `862`.

**Rotate:**

1. Contate o time SAP da Major (equipe ativy@majornutricao.com.br) e peça pra
   trocar a senha do user `862`.
2. **Nova senha ≠ `1234`**. Forte, 16+ chars, não reutilizada.
3. Atualize o secret AWS (novo secret, os antigos ficam nos launch.json
   que agora não existem):
   ```bash
   aws --profile tecnoepec-dev secretsmanager create-secret \
     --name development.TEPVENDAS_SAP_B1_MAJOR \
     --secret-string '{"username":"862","password":"NOVA_SENHA","baseUrl":"https://b1.ativy.com:50211","companyDb":"SBOMJM2016_TST"}' \
     --region us-east-1
   ```
4. Passe pra task ECS via env — o código já lê de env vars
   `MAJOR_SAP_B1_PASSWORD` etc, então adicione ao task-def:
   ```yaml
   secrets:
     - name: MAJOR_SAP_B1_PASSWORD
       valueFrom: arn:aws:secretsmanager:...:development.TEPVENDAS_SAP_B1_MAJOR:password::
   ```

---

## 4. Certificados Apple `G3Z2AJ7B5K` e `HV4U8VLWMU` — ALTO

**O que expõe:** capacidade de assinar apps em nome da conta Apple Developer.
Alguém com o `.p12` mais a senha (também estava no repo?) pode publicar app
fake com seu Team ID.

**Revogar:**

1. https://developer.apple.com/account/resources/certificates/list
2. Localize os certs pelos IDs acima (dev + distribution) e clique **Revoke**
3. Gere novos:
   - **Development** → tipo "Apple Development", CSR via Keychain no seu Mac
   - **Distribution** → tipo "Apple Distribution" (App Store)
4. Baixe, converta pra `.p12` no Keychain (Export → cifre com senha forte)
5. Guarde os `.p12` **fora do repo**: 1Password vault "TEP Vendas iOS" +
   backup em drive pessoal
6. Provisioning profiles antigos ficam inválidos — regenerar em Certificates,
   Identifiers & Profiles → Profiles

Não precisa mexer no que já foi publicado (App Store aceita apps assinados
com cert revogado — só bloqueia novas builds).

---

## Confirmação

Depois de completar as 4:

1. Rode o smoke test:
   ```bash
   # Login backoffice, envio de push, imagem no Firebase Storage, sync do SAP
   ```
2. Marca no `gestao-segredos.md` "Incidentes históricos" a coluna Remediação
   com **"(a) removido código + (b) rotate concluído set/2026-XX-XX"**
3. Encerra o incidente.

Se demorar mais de 2 semanas pra completar, o ideal é assumir que os
segredos foram usados por terceiros e monitorar CloudTrail / Firebase Audit
Logs por chamadas anômalas nesses ~15 dias.
