# Mobile Android — APK via Capacitor

App Android nativo que empacota o **Coletor Virtual** (React SPA) como APK
instalável. Roda offline (assets locais), conecta no backend via REST por IP
da LAN, usa scanner físico/câmera nativamente.

> **Estrutura monorepo**: o app Capacitor vive em `mobile/` (independente do
> `web/`). O `capacitor.config.ts` aponta `webDir: '../web/dist'`, então o
> `cap sync` puxa o build do React de lá. Scripts em `mobile/package.json`
> (`npm run sync`, `npm run android:debug`).

## Stack

| Camada | Tech |
|---|---|
| Shell nativo | **Capacitor 7** (Ionic/Outsystems) — WebView nativo + bridge JS↔Java |
| UI | React 18 + Vite (mesmo código do `web/`) |
| Scanner | **@capacitor-mlkit/barcode-scanning** (Google ML Kit) + DataWedge (Compex/Zebra) |
| Storage local | **@capacitor/preferences** (SharedPreferences nativo) |
| HTTP | axios (mesmo do web) |
| Build | Gradle 8 (Android Studio ou linha de comando) |

## Arquitetura

```
APK Android
├─ MainActivity (Capacitor WebView)
│   └─ assets/public/index.html        ← build do web/dist (offline)
│       └─ React SPA (TanStack Query, etc)
│           ├─ /api/v1/* → http://<IP_BACKEND>:8080  ← LAN
│           └─ scanner: ML Kit (camera) ou DataWedge intent (físico)
└─ Plugins nativos:
    ├─ BarcodeScanner (ML Kit)
    └─ Preferences (storage)
```

## Setup inicial (uma vez)

### 1. Pré-requisitos

- **Node 20+** (já tem)
- **JDK 17** — https://adoptium.net (versão LTS)
- **Android Studio** (vem com SDK) — https://developer.android.com/studio
  - Durante install marque: Android SDK Platform 34+ · Build Tools · Platform-Tools

Confirma:
```bat
java -version    REM precisa 17.x
adb --version    REM vem com SDK Platform-Tools
```

### 2. `local.properties` do projeto Android

Crie `web/android/local.properties` apontando pro Android SDK:

```properties
sdk.dir=C:\\Users\\SEU_USUARIO\\AppData\\Local\\Android\\Sdk
```

(O caminho típico no Windows é esse; ajuste se instalou em outro lugar.)

### 3. Build do web (gera os assets do APK)

```bat
cd C:\Projetos\Coletor_2001\web
npm run build
npx cap sync android
```

`cap sync` copia `dist/` pra `android/app/src/main/assets/public/` + atualiza
plugins.

## Build do APK

### Modo debug (rápido, pra testar)

```bat
REM Atalho via npm script (no diretório mobile/):
cd C:\Projetos\Coletor_2001\mobile
npm run android:debug

REM Ou manualmente:
cd mobile
npm run sync                       REM build do web + cap sync android
cd android
gradlew.bat assembleDebug
```

APK gerado em:
```
mobile\android\app\build\outputs\apk\debug\app-debug.apk
```

Instalação no aparelho (USB com modo desenvolvedor ON + USB debugging):
```bat
adb install -r app\build\outputs\apk\debug\app-debug.apk
```

### Modo release (pra distribuição, assinado)

1. **Gera keystore** (uma vez):
```bat
keytool -genkey -v -keystore coletor2001.keystore -alias coletor2001 ^
        -keyalg RSA -keysize 2048 -validity 10000
```
Guarde a senha — sem ela não dá pra publicar atualização futura no mesmo app.

2. **Configura `android/app/build.gradle`** (seção `signingConfigs`) com path
do keystore + senha.

3. **Build**:
```bat
gradlew.bat assembleRelease
```

APK em `android\app\build\outputs\apk\release\app-release.apk`.

## Configuração do backend

O APK precisa saber o IP do backend. Tem 2 abordagens:

### A. URL hardcoded em build time (mais simples)

Edite `web/src/api/client.ts` antes do build:
```ts
const BASE_URL = "http://192.168.X.X:8080/api/v1";  // IP do servidor prod
```

Depois `npm run build && npx cap sync android && gradlew assembleDebug`.

### B. Telas de configuração no app

Adicione tela inicial onde o operador digita o IP do servidor (já tem o
pareamento por token de dispositivo — basta ampliar pra incluir host).
Salva em `@capacitor/preferences` e o axios usa.

Implementação pendente — recomendado pra distribuir pra múltiplas lojas.

## Updates do app

Quando você muda o React (web/), o APK precisa rebuildar:

```bat
cd web
npm run build
npx cap sync android
cd android
gradlew.bat assembleDebug
adb install -r app\build\outputs\apk\debug\app-debug.apk
```

**Update over-the-air** (futuro): pode usar `@capacitor/live-updates` ou
servir o `dist/` direto do servidor backend (`server.url` no
`capacitor.config.ts`) — quando o operador abre o app, baixa a versão nova
sem reinstalar APK.

## Scanner

### ML Kit (qualquer Android)

Já cabeada via `web/src/utils/scanner.ts`:

```ts
import { escanearCamera, isNativoCapacitor } from "@/utils/scanner";

if (isNativoCapacitor()) {
  const codigo = await escanearCamera();  // abre câmera, retorna o código
}
```

### Scanner físico Compex/Zebra (DataWedge)

DataWedge no aparelho envia o código via intent broadcast. O `AndroidManifest.xml`
já tem o `<queries>` autorizado. Pra capturar dentro do React, adicione um
listener nativo (futuro: plugin custom OU usar `keyboard wedge mode` que o
DataWedge oferece — código vai pro input com foco como se digitado).

**Modo wedge** (mais simples): no painel DataWedge do aparelho, configure
o profile do app `com.empresa.coletor` pra **Keystroke output**. O código
bipado pelo scanner físico vai aparecer no input HTML que tiver foco +
ENTER automático.

## Splash + ícone

### Ícone

Substitua `android/app/src/main/res/mipmap-*/ic_launcher*.png` pelos seus.
Use o [Android Asset Studio](https://romannurik.github.io/AndroidAssetStudio/icons-launcher.html)
pra gerar todas as resoluções a partir de 1 PNG quadrado.

### Splash screen

Configurado em `capacitor.config.ts`:
```ts
plugins: {
  SplashScreen: {
    launchShowDuration: 1200,
    backgroundColor: '#006CB5',  // azul institucional
  },
}
```

Pra mudar a imagem do splash, gere PNGs e coloque em
`android/app/src/main/res/drawable-*/splash.png`.

## Versionamento do app

Em `android/app/build.gradle`:
```gradle
android {
    defaultConfig {
        versionCode 1        // INTEIRO — incremente a cada release na Play Store
        versionName "0.4.0"  // VISÍVEL — segue o release do backend/web
    }
}
```

## Troubleshooting

| Sintoma | Causa | Fix |
|---|---|---|
| `SDK location not found` | Falta `local.properties` | Crie com `sdk.dir=...` |
| `Java version mismatch` | JDK 11 ou outro | Instale JDK 17 e seta `JAVA_HOME` |
| App abre branco | URL backend errada / sem rede | Ver `chrome://inspect` (debug remoto), checar `client.ts` |
| Scanner não abre câmera | Sem permissão CAMERA | Aceitar prompt na 1ª vez |
| Erro `Cleartext HTTP traffic not permitted` | Backend HTTP puro | Adicione `cleartext: true` no `capacitor.config.ts` (não recomendado pra produção; use HTTPS) |
| `Module @capacitor/preferences not found` | Faltou `cap sync` | Rode `npx cap sync android` |

## Roadmap

- ⏳ Tela de configuração de IP do servidor (em vez de hardcoded)
- ⏳ Plugin custom pra DataWedge (capturar intent direto no JS)
- ⏳ Live updates do bundle JS (sem reinstalar APK)
- ⏳ APK assinado + upload pra Google Play interno
- ⏳ MDM integration (Knox / Android Enterprise) pra deploy massivo nos coletores
