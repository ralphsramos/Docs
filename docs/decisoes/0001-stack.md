# ADR-0001 — Escolha de stack

**Status:** Aceita
**Data:** 2026-05-25

## Contexto

O Coletor 2001 substitui o IS Collector externo e precisa de três frentes (API/integrações, painel web, app coletor) com integração nativa ao Winthor/Oracle.

## Decisão

| Camada | Escolha |
|--------|---------|
| Backend | **FastAPI** + SQLAlchemy 2.x async + Alembic |
| DB próprio | **PostgreSQL** (asyncpg) |
| ERP | **Oracle Winthor** via `oracledb` (thin mode, sem instant client) |
| Auth | JWT (HS256) + login contra `PCUSUARI`; perfis e permissões no Postgres |
| Painel Web | **React 18 + Vite + TypeScript** |
| Mobile | **Android nativo Kotlin** + Room (SQLite) + WorkManager |
| Tempo real | WebSocket (FastAPI) |

## Por quê

- **FastAPI**: async nativo casa com WebSocket + sync móvel; OpenAPI automático acelera contrato com web e mobile.
- **React + Vite**: equipe pode tocar UI complexa (datatables, filtros, dashboards) sem fricção de build; SPA isola painel da API.
- **Kotlin nativo**: coletores físicos (Mini20, Zebra, Honeywell) expõem o gatilho do scanner via Intent broadcast — bridge cross-platform geralmente exige plugin nativo de qualquer jeito, então melhor já ir direto. Room + WorkManager dão offline-first robusto.
- **oracledb thin**: dispensa Oracle Instant Client no servidor, simplifica deploy.
- **Login Winthor**: requisito do negócio (mesma credencial em toda operação), mas perfis ficam no Postgres para não poluir o ERP.

## Alternativas consideradas

- **Django + DRF**: admin pronto seria útil, mas Channels (WebSocket) e ORM síncrono atrapalhariam o fluxo tempo-real.
- **Jinja+HTMX**: simples de servir, mas painéis do IS Collector têm muita interação (filtros, modais, relatórios) — SPA escala melhor.
- **Flutter / RN**: menos atrito multi-plataforma, mas hardware-específico (scanner) acabaria forçando bridge nativa.

## Consequências

- Dois ambientes de build (Python + Node).
- Necessário tooling para alinhar contratos (OpenAPI → client TS gerado).
- Equipe precisa de skill Kotlin/Android para mobile.
