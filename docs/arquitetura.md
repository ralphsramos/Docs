# Arquitetura — Coletor 2001

## Visão geral

```
  Painel Admin (web)         App coletor (Android — em bootstrap)
  Coletor Virtual            Compex / Zebra com scanner
   │                              │
   │  JWT (HS256)                 │  X-Device-Token (9 dígitos)
   ▼                              ▼
            ┌─────────────────────────────────────────────────────┐
            │                  Backend (FastAPI)                  │
            │                                                     │
  REST + WS │   ┌──────────┐   ┌──────────┐   ┌─────────────┐   │  oracledb (thin)
  ──────────┼──▶│ Routers  │──▶│ Services │──▶│ Repositórios├───┼──────────────▶ Winthor (Oracle)
            │   └──────────┘   └──────────┘   └──────┬──────┘   │  ↑ APScheduler
            │   ↓                                    │           │     diário (02h
            │  ErrorLoggingMiddleware                ▼           │     configurável
            │  (error_logs)               ┌───────────────┐      │     pela UI)
            │                             │  SQLAlchemy   │      │
            │                             └───────┬───────┘      │
            └─────────────────────────────────────┼──────────────┘
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │   PostgreSQL     │
                                        │  24 tabelas      │
                                        └──────────────────┘
```

## Camadas do backend

- **`app/api/`** — Routers FastAPI. Versionados em `v1/`. Lógica mínima — só orquestram serviços e validam input/output via schemas Pydantic.
- **`app/schemas/`** — DTOs Pydantic (request/response). Separados dos models SQLAlchemy.
- **`app/services/`** — Regras de negócio (não conhecem FastAPI; recebem sessão e parâmetros). Inclui `dispositivo_log_service` (helper de auditoria), `export_service` (gera TXT/Excel/PDF), `auth_service`, `coleta_avulsa_service`.
- **`app/models/`** — Modelos SQLAlchemy (PostgreSQL) — ver [`modelo-dados.md`](modelo-dados.md).
- **`app/db/`** — Sessões (`session.py` Postgres async, `oracle.py` Winthor) e `Base` declarativa.
- **`app/integrations/winthor/`** — Adapters Oracle/API/Stub com interface comum (`base.py`).
- **`app/core/`** — Configuração (`config.py`), segurança JWT (`security.py`), logging estruturado, **`middleware.py`** com `ErrorLoggingMiddleware`.
- **`app/jobs/`** — Tarefas agendadas: `carga_produtos.py` (sync cadastro/estoque), `agendador.py` (APScheduler).

## Endpoints (release 0.3.0, ~50 endpoints únicos)

| Grupo | Endpoints |
|-------|-----------|
| **Sistema** | `GET /health` |
| **Auth** | `POST /auth/login`, `GET /auth/me` |
| **Usuários** | `GET/POST/PATCH /usuarios/`, `GET /usuarios/winthor`, `POST/DELETE /usuarios/{id}/atribuicoes(/{aid})` |
| **Perfis** | `GET/POST/PATCH/DELETE /perfis/`, `GET /perfis/permissoes` |
| **Filiais** | `GET /filiais/`, `POST /filiais/sincronizar` |
| **Produtos** | `GET /produtos/`, `GET /produtos/resumo`, `GET /produtos/por-filial`, `GET /produtos/sync-logs`, `POST /produtos/sincronizar/{cadastro,estoque/{id}}`, `GET /produtos/carga/{filial_id}` |
| **Agendador** | `GET /produtos/agendador/{status,config}`, `PUT /produtos/agendador/config`, `POST /produtos/agendador/rodar-agora` |
| **Dispositivos (admin)** | `GET /dispositivos/`, `GET /dispositivos/resumo`, `POST /dispositivos/`, `PATCH /dispositivos/{id}`, `POST /dispositivos/{id}/rotacionar-token`, `DELETE /dispositivos/{id}`, `GET /dispositivos/{id}/logs` |
| **Device (app)** | `POST /device/pair`, `GET /device/me`, `GET /device/carga`, `GET /device/display-config`, `POST /device/leituras`, `POST /device/coletas/arquivar`, `POST /device/logout`, `POST /device/login-operador`, `POST /device/logout-operador` |
| **Coleta Avulsa (admin)** | `GET /coletas-avulsas/`, `GET/PUT /coletas-avulsas/display-config` |
| **Coletas Arquivadas** | `GET /coletas-arquivadas/`, `GET /coletas-arquivadas/resumo`, `GET /coletas-arquivadas/{uuid}`, `DELETE /coletas-arquivadas/{uuid}`, `POST /coletas-arquivadas/bulk-delete` |
| **Exportação** | `GET /export-templates/`, `GET /export-templates/colunas-disponiveis`, `POST /export-templates/`, `PATCH /export-templates/{id}`, `DELETE /export-templates/{id}`, `POST /export-templates/{id}/usar/{uuid}` |
| **QA** | `GET /qa/{summary,logs,logs/{id}}`, `DELETE /qa/logs?mais_antigos_que_dias=N` |

Swagger interativo em http://localhost:8000/docs.

### Convenção de autenticação

| Prefixo | Autenticação | Quem usa |
|---------|--------------|----------|
| `/api/v1/auth/*` | Sem (login) | Painel web (1× pra obter JWT) |
| `/api/v1/device/*` | `X-Device-Token` (9 dígitos) | App coletor (Android, Coletor Virtual) |
| Demais `/api/v1/*` | JWT Bearer | Painel web (admin) |

## Banco PostgreSQL

24 tabelas operacionais. Ver [`modelo-dados.md`](modelo-dados.md).

Princípios:
- `created_at`, `updated_at`, `atualizado_em` onde faz sentido (timestamptz).
- FKs com `ON DELETE` explícito (`CASCADE` / `RESTRICT` / `SET NULL`) conforme semântica.
- Índices em campos consultados (`status`, `filial_id`, `path`, `winthor_login`).
- Constraints de integridade onde possível (ex.: `chk_acesso_total_xor_filial` em `usuarios_perfis`).

## Integração Winthor

Ver [`integracao-winthor.md`](integracao-winthor.md). Resumo:

- **Login**: procedure `PROC_AUTENTICACAO` valida a credencial dentro do ERP; o backend não lê nem armazena senha.
- **Cadastro de produto**: `PCPRODUT` + `PCEMBALAGEM` agrupados via LISTAGG.
- **Estoque**: `PCEST` filtrado por filial e por `QTESTGER<>0 OR QTRESERV<>0 OR QTBLOQUEADA<>0`.
- **Filiais**: `PCFILIAL` filtrado por `WINTHOR_FILIAIS_PERMITIDAS` (env `1..8`). Sem coluna `BLOQUEIO` nessa instalação.

A camada `integrations/winthor/` expõe interface única (`WinthorAdapter`) com 3 implementações:
- `OracleAdapter` — produção
- `ApiAdapter` — fallback REST (placeholder)
- `StubAdapter` — testes/dev

Selecionado por `WINTHOR_DRIVER=oracle|api|stub`.

## Permissões & escopo de filial

Ver [`permissoes-e-filiais.md`](permissoes-e-filiais.md). Resumo:

1. **Auth Winthor** valida senha
2. **Conta ativa no Coletor 2001** (`usuarios.ativo=true`)
3. **Perfil + filial** decide o que pode fazer e onde

Helper `filiais_permitidas(user)` em `app/api/deps.py` retorna `None` (vê tudo) pra admin ou quem tem `acesso_total`, senão lista de `filial_id`s. Aplicado em todas as rotas operacionais.

## Sincronização Winthor → Postgres (cargas)

Dois fluxos:

**Cadastro global** (`sincronizar_cadastro`):
- Lê `PCPRODUT` + LISTAGG `PCEMBALAGEM`
- Upsert em batches de 1000 (escala pros ~234k produtos da empresa)
- Dedup defensivo (codigo_principal / codigo_barras)
- Truncate defensivo de VARCHARs (reporta `N truncados` no SyncLog)

**Estoque por filial** (`sincronizar_estoque_filial`):
- Lê `PCEST WHERE CODFILIAL=:f AND (QTESTGER<>0 OR ...)`
- Reset + insert em batches (snapshot atômico por filial)
- Filtra produtos que não existem no cadastro local (reporta `N descartados`)

Cada execução grava um `SyncLog` com `tipo, status, duracao_ms, qtd_*, mensagem`.

## Agendamento

**APScheduler** iniciado no `lifespan` do FastAPI. Job único `sync_winthor_diaria` configurado por cron.

Configuração efetiva = **`app_config` (DB) sobrepõe `.env`**:
- `sync_agendador_ativo` (bool)
- `sync_cron_hour` (0-23)
- `sync_cron_minute` (0-59)
- `tz_agendador` (ex: `America/Sao_Paulo`)

Mudanças via `PUT /produtos/agendador/config` reagendam o job na hora (`reschedule_job` no APScheduler), sem precisar restart.

## Sincronização mobile (offline-first)

App coletor (Android nativo em bootstrap; web simulado funcional em `/coletor-virtual`) consome:

- `POST /device/pair` (uma única vez por instalação — registra a sessão e telemetria)
- `GET /device/carga` (uma chamada retorna catálogo + saldos + EANs por produto, gzip automático)
- `POST /device/leituras` (batch idempotente por `(dispositivo_id, uuid_local)`)
- `POST /device/coletas/arquivar` (finaliza a coleta — grava em `coletas_arquivadas` + payload JSONB)

Tudo autenticado por `X-Device-Token` (9 dígitos numéricos gerados no `/dispositivos`).

Estratégia offline (Android — release 0.4.0):
1. App baixa carga 1× por dia (manualmente ou cron interno).
2. Armazena em SQLite local (Room) — bipagem 100% offline.
3. Leituras vão pra fila em `leituras_local`.
4. `WorkManager` envia em batch via `POST /device/leituras`.
5. Backend faz upsert idempotente por `(dispositivo_id, uuid_local)`.

## Autenticação detalhada

**Web (admin):**
- `POST /auth/login` → `PROC_AUTENTICACAO` → cria/atualiza espelho em `usuarios` (Postgres) → JWT (access 30min + refresh 7 dias).
- `winthor_codfilial` (espelhado de `PCEMPR.CODFILIAL`) é sugerido como padrão na UI de vínculo.
- Se o admin desativar o usuário no painel, próximo login retorna 401 mesmo que Winthor aceite — não reativa automaticamente.

**App coletor (mobile):**
- 2 níveis: **token do dispositivo** (gerado pelo admin no painel, vale enquanto o dispositivo estiver ativo) + **login do operador** (Winthor, valida via `PROC_AUTENTICACAO` e fica preso em `dispositivos.operador_login_atual` até `/device/logout-operador`).
- Todo evento gera linha em `dispositivo_logs` (auditoria append-only).

## Auditoria de dispositivos

Tabela `dispositivo_logs` (JSONB de detalhes, índices por dispositivo/data/evento) captura **10 tipos de eventos**:

| Evento | Quando |
|--------|--------|
| `pareamento` | `POST /device/pair` |
| `sync` | `GET /device/carga` |
| `leituras_batch` | `POST /device/leituras` |
| `arquivar_coleta` | `POST /device/coletas/arquivar` (também marcado quando admin DELETA a coleta) |
| `logout` | `POST /device/logout` (operador digitou SIM no app) |
| `login_operador` | `POST /device/login-operador` OK |
| `login_operador_falhou` | login com senha errada (sem expor a senha) |
| `logout_operador` | `POST /device/logout-operador` |
| `token_rotacionado` | admin gerou novo token |
| `ativacao` | admin ativou/desativou |

Painel admin: botão **📋 Log** em cada linha de `/dispositivos` abre drawer lateral com timeline auto-refresh (10s).

## Sistema de exportação (TXT / Excel / PDF)

Tabela `export_templates` guarda layouts salvos (públicos ou privados) com config JSONB. Cada coleta arquivada pode ser exportada usando qualquer template do tipo correspondente:

- **TXT** (puro Python): config completa — modo (consolidado/linha-a-linha), separador (`;` `,` `\t` `qtd_chars:N` `custom:X`), colunas ordenáveis, header, prefixo, decimal config.
- **Excel** (`openpyxl`): cabeçalho azul `#006CB5`, descrições em vermelho `#EC3237`, zebra cinza `#EBEBEB`, linha de total. Config simplificada (modo + decimal — colunas fixas).
- **PDF** (`reportlab`): layout fiel ao mockup IS Collector — A4 com logo textual, bloco direito com metadados, divisória azul, tabela 4 colunas com descrições vermelhas, total mesclado. 2 modos (consolidado/linha-a-linha).

Endpoint `POST /export-templates/{id}/usar/{uuid}` gera o arquivo e devolve como `Response` com `Content-Disposition: attachment`.

Página `/export-templates` permite editar a config completa de qualquer template (inclusive os defaults) via PATCH.

## Impressão de etiquetas (módulo Impressão)

Stack independente que se conecta direto na **rede LAN das impressoras térmicas**
(Argox/Zebra/Elgin) via TCP raw na porta 9100, sem driver, CUPS ou cloud.

- **Linguagem**: ZPL II (Zebra nativo + emulação Argox PPLZ / Elgin ZB). Cobertura
  estimada ~95% do parque sem adaptação.
- **Render**: Jinja2 substitui `{{ campo }}` no template do layout antes do envio.
  Editor visual react-konva produz `elementos` JSON, backend converte em ZPL.
- **Preview**: Labelary API (api.labelary.com) renderiza ZPL→PNG ao vivo no editor
  e na aba ⚡ Direta.
- **Transporte**: `asyncio.open_connection(host, 9100)` — não bloqueia event loop.
- **Worker fila** (APScheduler 30s): processa jobs `imprimir_depois` com retry
  exponencial (1s/5s/30s/2min/10min, max 5 tentativas).
- **Heartbeat** (60s): TCP ping em cada impressora ativa, atualiza
  `online_em` / `ultima_falha_em`. UI mostra bolinha verde/cinza.
- **Idempotência mobile**: cada batch tem `uuid_local` salvo no JSONB do job —
  retransmissão não duplica.

4 tabelas: `impressoras`, `impressao_layouts`, `impressao_jobs`,
`impressao_jobs_log`. 5 abas no admin (`⚡ Direta` / `📋 Coletas` / `🏷️ Layouts`
/ `🖨️ Impressoras` / `📜 Log`), filtradas por permissões granulares
(`impressao.{aba}.ver`). Mobile tem MenuCard "🖨 ETIQUETAS" no Coletor Virtual.

Ver [`impressao.md`](impressao.md) pra detalhes completos.

## Tempo real (planejado — 0.5.0+)

WebSocket `ws/inventario/{id}` pra que cada bipe sincronizado dispare broadcast pros painéis subscritos. Atualmente as listas usam polling 2.5-15s via TanStack Query (`refetchInterval`).

## Observabilidade

- **Logs estruturados** JSON via `structlog` (campo `request_id` propagado por `contextvars`).
- **`error_logs` table** captura 5xx + 401/403 com traceback, payload (saneado de Authorization/Cookie), IP, user-agent.
- **`x-request-id`** header em toda resposta — rastreabilidade.
- Painel `/qa` com cards de resumo + tabela filtrável + modal de detalhe (ver [`qa.md`](qa.md)).

## Stack consolidada (release 0.3.0)

| Camada | Tecnologia |
|--------|------------|
| Backend | FastAPI 0.115 + SQLAlchemy 2.x async + Alembic |
| DB próprio | PostgreSQL 17 (asyncpg) |
| ERP | Oracle 19c (`oracledb` thin) |
| Auth web | JWT (HS256) + `PROC_AUTENTICACAO` |
| Auth mobile | `X-Device-Token` (9 dígitos) + login operador Winthor |
| Frontend | React 18 + Vite + TS + TanStack Query + React Router |
| Agendador | APScheduler 3.11 |
| Exportação | openpyxl (Excel) + reportlab (PDF) + StringIO puro Python (TXT) |
| Mobile | Kotlin nativo + Room + WorkManager (release 0.4.0); simulador web funcional em `/coletor-virtual` |
| Observabilidade | structlog + middleware custom + tabela `error_logs` + tabela `dispositivo_logs` |
| IDE | PyCharm com Run Configurations versionadas (`.idea/runConfigurations/`) |
