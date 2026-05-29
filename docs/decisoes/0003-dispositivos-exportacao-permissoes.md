# ADR-0003 — Dispositivos, exportação e permissões granulares

**Status:** Aceita
**Data:** 2026-05-25 (tarde/noite)
**Decisões consolidadas na release 0.3.0.**

## Contexto

A release 0.2.0 fechou cadastros e QA. Pra começar a operação real precisamos de:

1. **Quem é o coletor físico?** — Não dá pra confiar só em JWT de usuário (operador troca, dispositivo fica).
2. **O que aconteceu nesse coletor?** — Operador reclama de coleta perdida → como investigar?
3. **Como o admin entrega o resultado da coleta?** — TXT padrão WMS, planilha pro financeiro, PDF pra arquivo, formatos diferentes por cliente.
4. **Quem pode apagar uma coleta?** — Antes qualquer um com acesso podia; precisamos distinguir leitura, exportação e exclusão.

## Decisões

### 1. Dispositivo como entidade de primeira ordem

**Tabela `dispositivos`** (já existia) ganhou telemetria (`versao_app`, `plataforma`, `modelo`, `ultimo_ip`, `primeiro_pareamento_em`, `total_leituras`) + **sessão de operador** (`operador_login_atual`, `operador_login_em`).

**Token de 9 dígitos numéricos** (em vez dos 32 hex anteriores):
- Coletores físicos (Compex/Zebra Mini20) têm teclado **só numérico**.
- 1 bilhão de combinações (`secrets.randbelow(999_999_999)`) — espaço suficiente pra frota de centenas.
- Coluna mantém `String(64)` por compat com tokens antigos. Máscara virou `…1234`.

**Autenticação `X-Device-Token`** em todos `/api/v1/device/*` — separado do JWT de usuário. App nunca precisa de login Winthor pra operar; só pra **identificar o operador** (que vai ser propagado nas leituras).

### 2. Login do operador SEM emitir JWT

**Alternativa rejeitada:** emitir JWT curto pro operador e o app mandar nos headers.

**Decisão:** salvar `operador_login_atual` direto no `dispositivos`. Razões:
- O app já tem `X-Device-Token` — adicionar JWT seria 2 headers + lógica de refresh.
- Operador troca em horas (não dias) — não vale TTL longo.
- `/device/leituras` e `/device/coletas/arquivar` automaticamente atribuem ao operador atual sem precisar de payload extra.
- Logout simples: zera o campo. Pra trocar de operador, nem precisa desparear.

Login em si reusa `autenticar_e_sincronizar` (mesma função do painel) — `PROC_AUTENTICACAO` no Oracle.

### 3. Auditoria append-only (`dispositivo_logs`)

10 tipos de evento padronizados como constantes Python (`EVENTO_PAREAMENTO`, `EVENTO_SYNC`, ...) — não strings mágicas espalhadas pelo código.

**Helper centralizado** `registrar_evento(session, dispositivo, evento, detalhes, operador_login, ip)`:
- Caller não comita (services compõem várias operações).
- `detalhes` é JSONB livre — cada evento define seu próprio shape (ex.: `sync` guarda `{produtos, com_saldo, filial_id}`; `leituras_batch` guarda `{recebidas, aceitas, duplicadas}`).
- Login falho grava `login_operador_falhou` mas **não expõe a senha** (só o login tentado).

Mesmo após admin **deletar** uma coleta arquivada, o evento `arquivar_coleta {apagada: true, apagada_por: <admin>}` continua no log do dispositivo — auditoria preservada.

### 4. Coletas arquivadas em 2 tabelas (header + payload JSONB)

```
┌─────────────────────────┐       ┌──────────────────────────────────┐
│  coletas_arquivadas     │       │  coletas_arquivadas_payload      │
│  ─────────────────────  │ uuid  │  ─────────────────────────────   │
│  uuid (PK rastreio)     │ ◄───► │  uuid (PK FK CASCADE)            │
│  qtd_leituras: 1247     │       │  payload: jsonb {                │
│  soma: 8234.5           │       │    versao_schema: 1,             │
│  dispositivo / operador │       │    leituras: [ 1247 items ]      │
│  arquivada_em           │       │  }                               │
│  ── 200 bytes/linha ─── │       │  tamanho_bytes                   │
└─────────────────────────┘       └──────────────────────────────────┘
        ↑                                       ↑
        listagem /coletas-avulsas               drawer "Detalhes"
        (rápida: KPIs + filtros + bulk)         (lazy-load 1× quando abrir)
```

PostgreSQL TOAST comprime o JSONB automaticamente quando passa de ~2KB. Coleta típica (~100 leituras) fica em 5-10 KB; coleta grande (1000+) em ~50-100 KB.

### 5. Exportação como **template salvável** (não config inline)

Cada exportação podia ser configurada na hora e descartada. Decisão: persistir.

**Tabela `export_templates`** (`nome`, `tipo`, `publico`, `owner_id`, `config` JSONB, `eh_default`).

Vantagens:
- Operador clica "Carregar modelo" e pega o layout que o supervisor preparou → consistência.
- Migration cria 3 templates default públicos — admin tem algo pra editar de cara, não precisa criar do zero.
- "Salvar como novo modelo" no modal de download → admin descobre o layout certo iterativamente.
- Página dedicada `/export-templates` permite editar a config completa (não só renomear).

**Formato simplificado pro Excel:** modal só mostra Modo + Decimal + Não-coletados. Colunas/header são fixas. Quem quer customizar mais edita o template direto na página de layouts. Pra TXT o modal mostra tudo porque o user-base se preocupa muito com separador/colunas.

### 6. Geração com libs sem dependência de SO

- **TXT** — `io.StringIO` puro Python, encoding UTF-8 (sem BOM).
- **Excel** — `openpyxl` (puro Python).
- **PDF** — `reportlab` (puro Python, A4, fontes Helvetica embutidas).

Evita `weasyprint`/`libreoffice`/`wkhtmltopdf` — ambos precisam de libs C/Java pesadas e quebram em ambientes Windows corporativos.

Cores oficiais do **manual de marca da empresa** (`Mini_20manual.pdf`):
- Azul `#006CB5` (Pantone 3005 C) — primary
- Vermelho `#EC3237` (Pantone Red 032 C) — destaques (descrições)
- Cinza `#EBEBEB` — zebra de linhas, fundo de total
- Branco `#FFFFFF` — superfícies

### 7. Permissões granulares com **área (web/mobile/comum)**

Antes: 8 chaves planas (`coleta.criar`, `admin.usuarios`, etc).
Agora: **17 chaves agrupadas por área** + rótulo amigável em pt-BR.

Razões:
- UI do EditorPerfil consegue separar visualmente (3 cards: 🖥 Web · 📱 Mobile · 🔁 Comum) — admin não fica perdido scrollando 17 checkboxes em lista plana.
- Permissões `coletas_arquivadas.{ver,exportar,excluir}` resolvem o caso "supervisor pode ver e baixar mas não apagar" sem inventar perfil novo.
- "Marcar todas" por área pra perfis especializados (operador 100% mobile, admin 100% web).

Auto-migração: quem tinha `admin.coletas_arquivadas` ganhou `.ver` + `.exportar`; **só "admin" ganhou `.excluir`**.

UI esconde botões via `hook usePermissions` lendo cache do `useQuery(['me'])` — **segurança real é no backend** (`require_any_permission`), o frontend só evita oferecer opções que vão dar 403.

## Alternativas consideradas

- **JWT de operador com curta duração** — rejeitado (item 2 acima).
- **Token alfanumérico (32 chars) pro dispositivo** — rejeitado (item 1: teclado físico).
- **Audit log único agregando TODOS os eventos do sistema** — rejeitado: `dispositivo_logs` é específico do contexto do dispositivo (filtro/timeline naturais); erros de aplicação já estão em `error_logs`; sync ficam em `sync_logs`. Cada tipo de evento tem seu lugar.
- **Salvar payload da coleta em coluna `bytea` (binário comprimido)** — rejeitado: JSONB já é comprimido pelo TOAST e mantém queryable se precisarmos extrair coisa específica depois (`payload->>'operador_login'`).
- **Gerar arquivos com Pandas + xlsxwriter** — Pandas adiciona ~30MB de deps. Excel das coletas é simples (header + linhas + total), `openpyxl` direto basta.
- **Endpoint `/export-templates/{id}/preview` que não persiste** — adiar pra 0.8.0. Hoje, baixar sem template salvo cria um `_rascunho_<timestamp>` privado (funciona mas suja a tabela com lixo se admin não limpar).

## Consequências

**Bom:**
- App é trivial de instalar (digita 9 dígitos uma vez, login a cada turno).
- Toda ação do operador é rastreável (drawer "📋 Log" no admin).
- Coletas grandes não engasgam a listagem (header rápido, payload lazy).
- Admin pode padronizar layouts por cliente (templates públicos).
- Excel e PDF têm visual consistente com a marca da empresa.
- Apenas admin pode apagar — supervisor consegue ver e baixar.

**Cuidado:**
- `_rascunho_*` templates acumulam em `export_templates` — precisamos limpar periodicamente OU implementar o endpoint `/preview`.
- `dispositivo_logs` cresce indefinidamente — sem rotina de purga (futuro: `DELETE WHERE criado_em < now() - 90 days`).
- Coluna `token` ainda é `String(64)` por compat — quando confirmarmos que não há mais nenhum token hex em uso, encolher pra `String(9)` numa migration futura.
- O endpoint `POST /export-templates/{id}/usar/{uuid}` é síncrono — coleta com 5000 leituras + PDF formatado pode levar 5-10s. Aceitável por enquanto; se virar problema, mover pra background job + endpoint de polling.
