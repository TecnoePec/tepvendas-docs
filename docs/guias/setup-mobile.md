# Setup Mobile

Ambiente pra rodar o **`tep_vendas_mobile`** (Flutter 3.38 / Dart 3.9) localmente em emulador Android ou iOS.

## Pré-requisitos

- **Flutter SDK 3.38.9** (versão travada no CI). Instala via [fvm](https://fvm.app/) pra manter isolado:
    ```bash
    dart pub global activate fvm
    fvm install 3.38.9
    fvm use 3.38.9
    ```
- **Android:** Android Studio + emulador AVD (arm64 no Apple Silicon, x86_64 em Intel/AMD).
- **iOS:** Xcode 15+ + Simulator (só macOS).
- Backend rodando (localmente ou dev via VPN) — o app aponta pra `https://tepvendas-api.tecnoepec.com.br` por padrão.

## Clonar + pub get

```bash
git clone git@github.com:TecnoePec/tep_vendas_mobile.git
cd tep_vendas_mobile
flutter pub get
```

## Configuração de ambiente

Cria `.env` na raiz (não versionado):

```env
BASE_URL_API=https://tepvendas-api.tecnoepec.com.br
ENCRYPTION_KEY=<qualquer-string-32-chars>
SYNC_PERIODICITY=120
```

O `CustomRemoteConfig` lê primeiro o Firebase Remote Config; se não conseguir, cai pro `.env` via `EnvConfig`.

!!! info "Mapa não precisa mais de chave"
    A tela `ClientAddressMap` usa `flutter_map` (OpenStreetMap) + OSRM público — sem chave Google Maps. Se você achar referências a `GOOGLE_API_KEY` num commit antigo, ignore: foi removido.

## Rodar

**Emulador Android:**
```bash
open -a Simulator                       # se iOS
flutter emulators --launch <avd-name>   # se Android
flutter run
```

**Build APK debug:**
```bash
flutter build apk --debug --target-platform android-arm64
adb install build/app/outputs/flutter-apk/app-debug.apk
```

**Build release (só CI):**
```bash
flutter build apk --release --dart-define=ENV=production
```

## Login inicial (dev)

Após backend rodar `SeedTep`:

- Email: `dev@tep.com.br`
- Senha: definida no `SeedTep.SeedUsersAsync()` (procurar `crypto.Execute("...")`)

Fixture semeada (`SeedFixtures`):

- Cliente: **Juliano Menezes** (`CLI-JLM-01`) com endereços delivery + billing em Goiânia/GO
- CD: **Major Nutrição Animal - Goianira** (`CD-MAJOR-01`) com endereço em Goianira/GO
- DCCA: 30 km entre os dois

Ou seja: dá pra abrir o app e criar um orçamento end-to-end sem tocar em nenhum cadastro.

## Testes

```bash
flutter analyze --no-fatal-infos --no-fatal-warnings
flutter test
```

Suite atual: **530+ testes** unit + widget.

## Troubleshooting

**"No Android SDK found" no build:** `flutter doctor` reporta o problema. Instala Android SDK via Android Studio ou aponta `ANDROID_HOME`.

**Emulador arm64 crasha com "libflutter.so not found":** você buildou APK pra x86_64. Rebuilda com `--target-platform android-arm64`.

**Wizard trava no step 2 com "Nenhuma condição de pagamento":** o Hive tem `PaymentCondition.Status=1` (Inactive) por sync antigo. `adb shell pm clear br.com.tecnoepec.tepvendas` + relogin força sync completo. Se persistir, checa se `SeedCatalog` do backend rodou (deve haver 8 PCs com Status=0 e 16 PPTs com Status=0).

**Mapa fica cinza:** verifica que o app está com a versão nova que usa `flutter_map`. Se `pubspec.yaml` ainda lista `google_maps_flutter`, o app está desatualizado — `git pull` + `flutter pub get`.
