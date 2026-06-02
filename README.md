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
| [`app-v18`](../../releases/tag/app-v18) | apk | **App 0.6.3** (code 18) — ⭐ atual. A tela de login bloqueada (update obrigatório) ganhou o botão **ATUALIZAR AGORA** — 1 toque baixa a nova versão e abre o instalador. |
| [`app-v17`](../../releases/tag/app-v17) | apk | **App 0.6.2** (code 17) — monitoramento remoto: o app envia seus **logs por dispositivo** a cada sincronismo (aba "Logs do app" em Dispositivos). _(substituída)_ |
| [`app-v16`](../../releases/tag/app-v16) | apk | **App 0.6.1** (code 16) — impressora Bluetooth com permissão em tempo de execução (plugin novo). _(substituída)_ |
| [`app-v15`](../../releases/tag/app-v15) | apk | **App 0.6.0** (code 15) — tema alinhado ao web + câmera híbrida (nativa + fallback) + login sempre ao deslogar. _(substituída)_ |
| [`app-v12`](../../releases/tag/app-v12) | apk | **App 0.5.7** (code 12) — impressão direta pelo celular: Servidor / Rede (TCP) / Bluetooth. _(substituída)_ |
| [`app-v8`](../../releases/tag/app-v8) | apk | **App 0.5.3** (code 8) — device usa URL da LAN; safe-area; Enter no login. _(substituída)_ |
| [`app-v5`](../../releases/tag/app-v5) | apk | **App 0.5.0** (code 5) — base OTA. _(substituída)_ |

> Versões intermediárias (codes 9–11, 13) foram etapas de teste e seguem
> acessíveis em [Releases](../../releases), mas saíram da tabela pra não poluir.

O fluxo completo está em [`docs/app-versionamento.md`](docs/app-versionamento.md).

> 🔒 **Segurança:** todos os APKs são assinados com a **mesma** chave de release.
> Instale binários vindos **apenas** deste repositório ou do servidor oficial da
> rede, e confira o **SHA-256** publicado em cada Release antes de instalar.
