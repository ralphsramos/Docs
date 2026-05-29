# ADR-0002 — Cargas Winthor → Postgres e agendamento

**Status:** Aceita
**Data:** 2026-05-25
**Substitui:** parte de ADR-0001 (escolha de stack — agendador era um TBD)

## Contexto

O Coletor 2001 precisa servir um app Android **offline-first**: o operador baixa a carga, opera horas sem internet, e sincroniza depois. Isso exige um **cache local do PCEST + PCPRODUT + PCEMBALAGEM** no Postgres, atualizado **ao menos uma vez por dia**.

Tentar consultar o Oracle em todo bipe seria inviável (latência VPN + lock no Winthor). E o app não pode falar Oracle direto por motivos de segurança/rede.

## Decisão

### 1. Sync em dois fluxos separados

- **`sincronizar_cadastro`** — `PCPRODUT + PCEMBALAGEM`, global (não tem filial). ~234k produtos. Demora 1-2 min.
- **`sincronizar_estoque_filial(filial_id)`** — `PCEST` filtrado por filial + por saldo não-zero. ~30k linhas por filial. Demora ~30s cada.

Separação: cadastro muda pouco (delta pode chegar a 100/dia); estoque muda muito (movimentação intra-dia). Sincronizar os dois com periodicidades diferentes no futuro é trivial com essa separação.

### 2. Auditoria em `sync_logs`

Cada execução grava status (`em_andamento → ok | falhou`), duração em ms, contagens (produtos/EANs/estoque) e mensagem (erros, avisos como "N truncados" ou "N descartados"). Permite responder "última sync OK foi quando?" e "esse produto não chegou na filial X porque?".

### 3. Cache local em `estoques_filial` (PK composta)

`(codigo_principal, filial_id) → quantidade, qt_reservada, qt_bloqueada, datas`. Snapshot atômico por filial: `DELETE + INSERT` em batches. Reset garantido (não tem produto fantasma de sync anterior).

### 4. APScheduler embutido no FastAPI

`AsyncIOScheduler` com cron diário. Job único `sync_winthor_diaria` chama `sincronizar_tudo_agendado()` (cadastro + estoque de TODAS as filiais sequencialmente).

### 5. Configuração editável pela UI

Tabela `app_config(chave, valor)` sobrepõe `.env`. Endpoints `GET/PUT /produtos/agendador/config` aceitam ativo + hour + minute + tz; PUT **reagenda o trigger imediatamente** sem restart (`reschedule_job`).

Botão `⋯` no card "Agendador automático" abre modal de edição.

### 6. Endpoint dedicado `GET /produtos/carga/{filial_id}` pro Android

Uma chamada retorna catálogo + saldos + EANs por produto, gzip automático. Payload típico cai ~10x.

## Alternativas consideradas

- **Cron externo (Windows Task Scheduler ou systemd)** chamando `python -m app.jobs.carga_produtos`: mais simples mas dificulta UI de edição + duplica gerenciamento. APScheduler dentro do app dá uma única fonte da verdade.
- **NDJSON streaming na carga Android**: reduziria pico de memória, mas complica cliente. Adiar até virar problema real.
- **Tabela única produtos+estoque desnormalizada**: rejeitado — saldo varia por filial, produto é global. Manter normalizado.
- **Sync incremental por `DTULTALTER`**: implementado como parâmetro opcional (`alterados_desde`), mas o agendador usa sync completo por enquanto (simples e robusto pra escala atual).

## Consequências

- **Bom**: app Android tem fonte estável (Postgres local); UI da operação não trava com lentidão do Oracle; histórico permite debug.
- **Bom**: configuração de horário sem deploy.
- **Cuidado**: sync completo a cada 24h significa que o app pode mostrar saldo defasado em até 24h. Se virar problema, encurtar pra 4x/dia ou implementar sync incremental por delta.
- **Cuidado**: APScheduler é in-process — se o uvicorn cair, não roda. Aceitável (uvicorn em produção tem supervisor). Pra produção multi-instância, migrar pra job externo (Celery, RQ) ou eleger um nó "schedule leader".
