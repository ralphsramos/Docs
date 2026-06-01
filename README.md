# Coletor 2001 — Docs & Releases públicos

Repositório **público** de apoio ao app **Coletor 2001** — o sistema de coleta
de dados dos coletores das Lojas 2001, integrado ao Winthor.

O código-fonte vive no repositório **privado** `Coletor_2001`. Aqui fica só o que
precisa ser **público**: a documentação publicável e os **releases do app**.

## O que tem neste repositório

| Caminho | O que é |
|---|---|
| **[Releases](../../releases)** | Versões do app distribuídas como **APK** (asset baixável). Espelho/fallback do OTA — ver abaixo. |
| [`docs/`](docs) | Documentação do projeto (comece por [`docs/index.md`](docs/index.md)). |
| [`ROADMAP.md`](ROADMAP.md) | Planejamento e evolução do projeto, versão a versão. |

### Documentação em destaque

- [`docs/index.md`](docs/index.md) — porta de entrada da documentação.
- [`docs/app-versionamento.md`](docs/app-versionamento.md) — versionamento e **OTA** do app.
- [`docs/arquitetura.md`](docs/arquitetura.md) — arquitetura (backend / web / mobile).
- [`docs/integracao-winthor.md`](docs/integracao-winthor.md) — integração com o Winthor.
- [`docs/mobile-android.md`](docs/mobile-android.md) — o app Android nos coletores.
- [`docs/permissoes-e-filiais.md`](docs/permissoes-e-filiais.md) · [`docs/acesso-via-rede.md`](docs/acesso-via-rede.md) — acesso, permissões e rede.

## Releases do app = fallback do OTA

Os coletores se atualizam preferencialmente pelo **servidor na LAN** (rápido e
privado). Quando estão **fora da rede**, caem para o **APK público** publicado
aqui em Releases — é o link que vai no campo `url_publica` de cada versão, no
painel admin. O app valida o **SHA-256** antes de aplicar a atualização.

| Versão | Tipo | Conteúdo |
|---|---|---|
| [`app-v7`](../../releases/tag/app-v7) | apk | **App 0.5.2** (code 7) — ⭐ atual. App nativo abre direto no fluxo de coletor (token → login do operador → menu), em tela cheia, sem o painel admin. |
| [`app-v6`](../../releases/tag/app-v6) | apk | **App 0.5.1** (code 6) — habilita HTTP cleartext p/ falar com o backend HTTP da LAN. _(substituída pela 0.5.2)_ |
| [`app-v5`](../../releases/tag/app-v5) | apk | **App 0.5.0** (code 5) — base de versionamento/OTA. _(substituída)_ |

O fluxo completo está em [`docs/app-versionamento.md`](docs/app-versionamento.md).

> 🔒 **Segurança:** todos os APKs são assinados com a **mesma** chave de release.
> Instale binários vindos **apenas** deste repositório ou do servidor oficial da
> rede, e confira o **SHA-256** publicado em cada Release antes de instalar.
