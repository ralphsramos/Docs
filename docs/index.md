# Documentação — Coletor 2001

Índice das docs por categoria. Cada arquivo é independente — leia o que
precisa.

## Arquitetura & Modelagem

- [`arquitetura.md`](arquitetura.md) — Camadas, fluxo de request, observabilidade
- [`modelo-dados.md`](modelo-dados.md) — Tabelas Postgres com PKs/FKs (visão geral)
- [`decisoes/`](decisoes/) — ADRs (Architecture Decision Records)

## Integração externa

- [`integracao-winthor.md`](integracao-winthor.md) — Oracle adapter, autenticação, cargas Winthor
- [`winthor-fluxos-operacionais.md`](winthor-fluxos-operacionais.md) — Procedures e APIs específicas (preço DE/POR, ajustes etc)

## Módulos operacionais

- [`inventario.md`](inventario.md) — Módulo Inventário FASE 1 (setores, áreas, dispositivos, carga 3 fontes, quantidade avariada)
- [`auditoria-az.md`](auditoria-az.md) — Auditoria A-Z (ciclos, compradores, fornecedores, conferências, consulta cross-ciclo, produtos não encontrados)
- [`impressao.md`](impressao.md) — Impressão de etiquetas térmicas (ZPL, Argox/Zebra/Elgin, editor visual, aba Direta, mobile)
- [`mobile-android.md`](mobile-android.md) — APK Android via Capacitor (build, ML Kit barcode, DataWedge, updates)
- [`app-versionamento.md`](app-versionamento.md) — Versionamento + atualização OTA do app (version.json, check-update, force-update, dashboard de frota, release)

## Segurança & operação

- [`permissoes-e-filiais.md`](permissoes-e-filiais.md) — 3 camadas de controle (perfil + filial + permissão), permissões de acesso aos módulos, conta de sistema "systems", logout que limpa a sessão, fluxos do dia a dia
- [`qa.md`](qa.md) — Como rodar testes, ler logs, testar login

## Dev

- [`dev-comandos.md`](dev-comandos.md) — Subir/parar serviços, scripts CLI, Run Configs PyCharm
- [`acesso-via-rede.md`](acesso-via-rede.md) — Backend + frontend acessíveis da LAN, máscaras (DNS interno, /etc/hosts, mDNS)
- [`dev-vs-prod.md`](dev-vs-prod.md) — Dev/Prod paralelos: diretórios isolados, bancos separados, scripts deploy.bat, rollback
- [`portas.md`](portas.md) — Range 8150-8160: alocação por serviço, como mudar, firewall

## Outras

- [`../README.md`](../README.md) — Overview + status atual
- [`../ROADMAP.md`](../ROADMAP.md) — Próximas entregas
- [`../CHANGELOG.md`](../CHANGELOG.md) — Histórico (Keep a Changelog)

## 🌐 Repo público de docs

Só este `docs/` + o `ROADMAP.md` são **espelhados** no repo público
[github.com/ralphsramos/Docs](https://github.com/ralphsramos/Docs). O
`CHANGELOG.md` e o `README.md` **ficam de fora** (contêm dados internos). O sync
acontece via `scripts/sync_docs.bat` — roda manualmente após mudanças
importantes. O menu lateral do app aponta pra esse mirror público (não pro repo
privado do projeto).

Setup uma vez:
```bash
git clone https://github.com/ralphsramos/Docs.git C:\Docs2001
```

Depois cada vez que quiser atualizar:
```bash
scripts\sync_docs.bat
```
