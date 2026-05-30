# Versionamento & Atualização do App (OTA)

Como o app mobile (Coletor 2001) é versionado, distribuído e atualizado nos
coletores — e como o painel acompanha a frota.

---

## 1. Fonte única de versão — `version.json`

A versão vive em **um só lugar**: `version.json` na raiz do projeto.

```json
{ "version_code": 5, "version_name": "0.5.0", "tipo": "apk", "obrigatoria": false, "notas": "..." }
```

- **`version_code`** (inteiro, monotônico) — a **fonte da verdade** pra decidir
  "tem atualização?". Comparação é SEMPRE por aqui, nunca por `version_name`
  (string: `"1.10" < "1.9"` daria errado).
- **`version_name`** — rótulo amigável (semver recomendado), exibido ao usuário.

Quem lê esse arquivo:
- **Gradle** (`mobile/android/app/build.gradle`) → `versionCode`/`versionName` do APK.
- **Vite** (`web/vite.config.ts`) → injeta `VITE_APP_VERSION_*`, lido por
  `web/src/utils/appVersion.ts` (menu do app + reporte ao backend).
- **Backend** → registra cada versão publicada na tabela `app_versoes`.

Assim o número do APK, o do menu e o do banco **nunca divergem**.

## 2. Tipos de atualização (híbrido)

| Tipo | O que atualiza | Experiência |
|---|---|---|
| **bundle** | só o JS/web (a maioria das mudanças) | silenciosa, sem prompt (via `@capgo/capacitor-updater`) |
| **apk** | app nativo completo (muda plugin/permissão/SDK) | o Android mostra o instalador (o usuário confirma) |

O dia a dia é **bundle**; APK só quando mexe na camada nativa.

## 3. De onde o app baixa

1. **Backend na LAN** (fonte primária) — rápido, privado, funciona sem internet.
   `GET /api/v1/app-versoes/{id}/download` serve o binário da pasta local
   `app_releases/`.
2. **GitHub Release** (espelho/fallback) — quando o coletor está fora da rede.
   O campo `url_publica` da versão guarda o link do asset.

O app valida o **SHA-256** (entregue pelo `/check-update`) antes de aplicar.

## 4. Onde ficam os binários

- **Local (máquina de build):** `app_releases/` — histórico completo de APKs/bundles.
  **NÃO** vai pro git (binários incham o repositório). Só o `index.json` e as
  `notas/*.md` são versionados.
- **GitHub:** a última versão sobe como **asset de Release** (`gh release`), não
  commitada no repo. As notas `.md` podem ir pro repo público de docs.

## 5. Fluxo de release (na máquina de build)

Pré-requisitos: JDK 21 + Android SDK + `mobile/release.keystore` +
`mobile/keystore.properties` (uma vez: `scripts/gerar_keystore.bat`).

> 🔑 **Todos os APKs usam a MESMA keystore.** Se a assinatura mudar, o Android
> recusa o update dos apps já instalados. Faça backup da keystore; sem ela não
> dá pra atualizar a frota existente.

```bat
python scripts/bump_version.py --name 0.6.0 --notas "O que mudou"
python scripts/release_app.py            REM builda APK assinado, sha256, index, notas
REM o script imprime os comandos pra: registrar no backend + gh release upload
```

`release_app.py` **não publica sozinho** — imprime os passos finais (upload no
backend + `gh release`) pra você revisar.

## 6. Como o app verifica e atualiza

1. Operador toca **SINCRONIZAR DADOS** (mesmo botão de produtos/impressoras).
2. Além de leituras + carga, o app chama `GET /device/check-update?version_code=N`.
3. O backend responde: `tem_update`, `obrigatoria`, `version_name`, `sha256`,
   `url_local` (LAN) + `url_publica` (GitHub), `notas`.
4. O menu mostra um **banner**:
   - **opcional** (azul): "Nova versão disponível" + botão **Atualizar agora**.
   - **obrigatória** (vermelho): além do banner, o **login do operador é
     bloqueado** (HTTP 426) até atualizar.
5. **Atualizar agora** → bundle aplica silenciosamente / APK abre o instalador.

A versão instalada aparece no **menu** (rodapé) e no **drawer** do app.

## 7. Controle de distribuição (painel)

Em **Dispositivos → Versões do app** o admin:
- torna uma versão **obrigatória** (força o update — bloqueia login),
- **retira** uma versão (rollback / "versão com problema" — para de ser
  oferecida no `/check-update` mesmo sendo a de maior code),
- vê notas, SHA-256, se o binário está na rede e o link público.

## 8. Dashboard de frota (painel)

Em **Dispositivos → Frota & versões**:
- **Cobertura de versão**: atualizados × desatualizados × sem versão reportada.
- **Distribuição por versão** (barras) e **cobertura por filial**.
- Por equipamento: **versão instalada** (com selo ⚠ desatualizado), **tempo de
  uso** (soma das sessões login→logout) e **quantidade de uso** (sessões +
  leituras).

Telemetria: `dispositivos.version_code_instalado`, `ultimo_check_update`,
`tempo_uso_segundos`, `total_sessoes` (alimentados no pair / check-update /
login-operador / logout-operador).

## 9. Vínculo do dispositivo (só admin desvincula)

O pareamento (rede → token → login do operador) fica **salvo no aparelho**. O
operador pode **trocar de operador** (logout do operador), mas **não** pode mais
desvincular o dispositivo — isso é exclusivo do **admin**, pelo painel
**Dispositivos** (rotacionar token = força novo pareamento; desativar; excluir).

## 10. Arquivos-chave

| Camada | Arquivo |
|---|---|
| Fonte de versão | `version.json` |
| Modelo/migration | `backend/app/models/app_version.py` · migration `d4a1b8e93c27` |
| Service | `backend/app/services/app_version_service.py` |
| API admin | `backend/app/api/v1/app_versoes.py` |
| API device | `backend/app/api/v1/device.py` (`/check-update`) |
| Dashboard | `backend/app/api/v1/dispositivos.py` (`/dashboard-versoes`) |
| App (versão/menu/banner) | `web/src/pages/ColetorVirtualPage.tsx` · `web/src/utils/appVersion.ts` |
| OTA apply | `web/src/utils/otaUpdate.ts` |
| Painel frota/versões | `web/src/pages/DispositivosPage.tsx` |
| Build/release | `scripts/bump_version.py` · `scripts/release_app.py` · `scripts/gerar_keystore.bat` |
