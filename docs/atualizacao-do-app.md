# Atualização do app (OTA)

> 🚧 **Em construção.**

O app se mantém atualizado **pela rede**, sem precisar baixar nada manualmente.

## Como funciona

- Ao **sincronizar**, o app verifica se há uma versão nova.
- Se houver, oferece **Atualizar agora** (algumas versões podem ser
  obrigatórias).
- O app **confere o SHA-256** do arquivo antes de instalar — se não bater, a
  instalação é **cancelada**.

## De onde vem a atualização

- **Dentro da rede:** direto do servidor (mais rápido).
- **Fora da rede:** do **[Releases](../../releases)** público (fallback), por
  HTTPS.

## Releases

Todas as versões publicadas ficam em **[Releases](../../releases)** — cada uma
com o **APK assinado** e o seu **SHA-256**.

> 🔒 Instale binários **apenas** do servidor oficial da sua rede ou deste
> repositório, e confira o SHA-256 publicado antes de instalar.
