# Modelo de Dados — PostgreSQL

24 tabelas operacionais (release 0.3.0). PKs `bigserial` salvo onde marcado.

```
identidade        cadastros          operação                 sync/QA/export
─────────         ─────────          ────────                 ──────────────
usuarios          filiais            inventarios              estoques_filial
perfis            dispositivos       setores                  sync_logs
permissoes        produtos           areas                    error_logs
perfis_           codigos_           inventario_operadores    app_config
  permissoes        relacionados     inventario_dispositivos  dispositivo_logs ★
usuarios_perfis                      leituras                 coletas_arquivadas ★
                                                              coletas_arquivadas_payload ★
                                                              export_templates ★
                                                              ★ = nova em 0.3.0
```

## Identidade & Acesso

### `usuarios`
| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| winthor_login | varchar(50) UNIQUE | `PCEMPR.USUARIOBD` upper |
| nome | varchar(120) | |
| email | varchar(120) | |
| **winthor_codfilial** | varchar(2) | **NOVO 0.2.0** — `PCEMPR.CODFILIAL`, sugestão default na UI de vínculo |
| ativo | bool | default true — se false, admin precisa reativar |
| ultimo_login | timestamptz | |
| created_at / updated_at | timestamptz | |

### `perfis`
| Campo | Tipo |
|-------|------|
| id | bigserial PK |
| nome | varchar(60) UNIQUE |
| descricao | varchar(255) |

### `permissoes`
| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| chave | varchar(80) UNIQUE — ex: `coletas_arquivadas.excluir` | |
| descricao | varchar(255) | |
| **area** | varchar(10) | **NOVO 0.3.0** — `web` / `mobile` / `comum` (UI agrupa por área) |
| **rotulo** | varchar(80) | **NOVO 0.3.0** — nome amigável em pt-BR (fallback: chave) |

**17 permissões em uso** (release 0.3.0):

| Chave | Área | O que libera |
|-------|------|--------------|
| `admin.usuarios` | web | Gerir usuários e perfis |
| `admin.dispositivos` | web | Gerir dispositivos e tokens |
| `admin.filiais` | web | Gerir filiais |
| `admin.produtos` | web | Gerir produtos / sincronização |
| `admin.coletas_arquivadas` | web | Ver coletas arquivadas (legado — preferir granulares abaixo) |
| `qa.ver` | web | Painel `/qa` |
| `coletas_arquivadas.ver` | web | Listar e ver detalhe das coletas arquivadas |
| `coletas_arquivadas.exportar` | web | Baixar TXT/Excel/PDF de coletas |
| `coletas_arquivadas.excluir` | web | Apagar coletas (**restrito ao admin**) |
| `inventario.efetivar` | web | Efetivar contagem no Winthor |
| `separacao.efetivar` | web | Efetivar separação no Winthor |
| `mobile.coleta_avulsa` | mobile | Coleta avulsa no app |
| `mobile.inventario` | mobile | Inventário no app |
| `mobile.conferencia` | mobile | Conferência no app |
| `mobile.separacao` | mobile | Separação no app |
| `mobile.arquivar_coleta` | mobile | Finalizar e arquivar coleta |
| `coleta.criar`, `inventario.criar`, `conferencia.criar`, `separacao.criar` | mobile | Versões legadas (compat) |

### `perfis_permissoes` (N:N)
`(perfil_id, permissao_id)` PK composta.

### `usuarios_perfis`
| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| usuario_id | bigint FK → usuarios | |
| perfil_id | bigint FK → perfis | |
| **filial_id** | bigint FK → filiais — **NULLABLE** | (XOR com acesso_total) |
| **acesso_total** | bool | **NOVO 0.2.0** — se true, ignora filial_id e vê todas |
| created_at / updated_at | timestamptz | |

**Constraint XOR (`chk_acesso_total_xor_filial`):**
```sql
(acesso_total = TRUE  AND filial_id IS NULL)
OR
(acesso_total = FALSE AND filial_id IS NOT NULL)
```

## Cadastros

### `filiais`
| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| codigo_winthor | int UNIQUE | `PCFILIAL.CODIGO` |
| nome | varchar(120) | |
| created_at / updated_at | timestamptz | |

### `dispositivos`
Expandido na 0.3.0 com telemetria + sessão de operador.

| Campo | Tipo | Notas |
|-------|------|-------|
| id | uuid PK | UUID v4 |
| apelido | varchar(60) | exibido no painel |
| token | varchar(64) UNIQUE | **0.3.0**: 9 dígitos numéricos (era 32 hex). Coluna mantém String(64) por compat. |
| filial_id | bigint FK | nullable |
| ativo | bool | liga/desliga pelo admin (sem deletar histórico) |
| **versao_app** | varchar(40) | **0.3.0** — reportado pelo app no pareamento |
| **plataforma** | varchar(20) | **0.3.0** — android / web-simulator / ios |
| **modelo** | varchar(60) | **0.3.0** — modelo do hardware reportado pelo app |
| **ultimo_ip** | varchar(45) | **0.3.0** |
| **primeiro_pareamento_em** | timestamptz | **0.3.0** |
| ultimo_sync | timestamptz | |
| **total_leituras** | int | **0.3.0** — contador acumulado |
| **operador_login_atual** | varchar(50) | **0.3.0** — operador logado no app (sessão preservada até logout) |
| **operador_login_em** | timestamptz | **0.3.0** |

### `dispositivo_logs` — **NOVO 0.3.0**
Auditoria append-only de eventos do dispositivo (migration `f1b8d9a2c4e5`).

| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| dispositivo_id | uuid FK → dispositivos | ON DELETE CASCADE |
| criado_em | timestamptz | server_default `now()` |
| evento | varchar(40) | indexado — ver tipos abaixo |
| detalhes | jsonb | livre — versão, ip, contagens, mensagem etc |
| operador_login | varchar(50) | quem disparou (operador logado ou admin) |
| ip | varchar(45) | |

**10 tipos de evento padronizados:**
`pareamento`, `sync`, `logout`, `login_operador`, `login_operador_falhou`, `logout_operador`, `leituras_batch`, `arquivar_coleta`, `token_rotacionado`, `ativacao`.

### `produtos`
| Campo | Tipo | Notas |
|-------|------|-------|
| codigo_principal | varchar(30) PK | `PCPRODUT.CODPROD` |
| descricao | varchar(255) | |
| **codigo_fabricante** | varchar(60) | **0.2.0** — `PCPRODUT.CODFAB` |
| **codigo_barras_principal** | varchar(60) | **0.2.0** — `PCPRODUT.CODAUXILIAR` |
| unidade | varchar(15) | `PCPRODUT.UNIDADE` |
| **unidade_master** | varchar(15) | **0.2.0** — `PCPRODUT.UNIDADEMASTER` |
| **codigo_ncm** | varchar(30) | **0.2.0** — `PCPRODUT.CODNCMEX` |
| **multiplo** | numeric(14,3) | **0.2.0** — `PCPRODUT.MULTIPLO` |
| atualizado_em | timestamptz | |

> **Removidos na 0.2.0:** `unidades_por_pallet`, `exige_info_extra` (eram do design IS Collector, sem origem no Winthor — voltam na release 0.7.0).

### `codigos_relacionados`
EANs adicionais do `PCEMBALAGEM`.

| Campo | Tipo |
|-------|------|
| codigo_barras | varchar(60) PK |
| codigo_principal | varchar(30) FK → produtos |

## Operação

### `inventarios`
Tabela polimórfica — serve para os 4 fluxos.

| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| filial_id | bigint FK | |
| nome | varchar(120) | |
| tipo | varchar(20) | `coleta_avulsa` / `inventario` / `conferencia` / `separacao` |
| status | varchar(20) | `planejamento` / `andamento` / `finalizado` |
| obriga_identificacao_operador | bool | |
| origem_dados | varchar(20) | `manual` / `oracle` / `api_winthor` |
| referencia_externa | varchar(60) | NF, pedido, etc. |
| data_execucao | date | |
| created_by | bigint FK → usuarios | |
| created_at / updated_at | timestamptz | |

### `setores`, `areas`
- `setores(id, inventario_id, nome)` — agrupador
- `areas(id, setor_id, numero_area, endereco, status_transmissao)` — onde a coleta é executada

### `inventario_operadores`, `inventario_dispositivos`
Restringem quem/o-que pode trabalhar no inventário (N:N).

### `leituras`
Linha de coleta — fonte da verdade.

| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| uuid_local | uuid | gerado pelo app — idempotência |
| inventario_id | bigint FK | |
| area_id | bigint FK | nullable |
| dispositivo_id | uuid FK | |
| operador_id | bigint FK → usuarios | nullable |
| codigo_lido | varchar(60) | EAN que entrou no scanner |
| codigo_principal | varchar(30) FK → produtos | resolvido (nullable se não bater) |
| quantidade | numeric(14,3) | |
| pallets | int | |
| info_extra | varchar(60) | lote/série (release 0.7.0) |
| data_hora_dispositivo | timestamptz | |
| data_hora_sync | timestamptz | |

**Constraint única:** `(dispositivo_id, uuid_local)` — bate-papo offline-first não duplica.

## Sync / Estoque

### `estoques_filial` — **NOVO 0.2.0**
Cache local do `PCEST` por filial.

| Campo | Tipo | Notas |
|-------|------|-------|
| codigo_principal | varchar(30) FK → produtos | |
| filial_id | bigint FK → filiais | |
| quantidade | numeric(14,3) | `PCEST.QTESTGER` |
| qt_reservada | numeric(14,3) | `PCEST.QTRESERV` |
| qt_bloqueada | numeric(14,3) | `PCEST.QTBLOQUEADA` |
| data_ultima_entrada | date | `PCEST.DTULTENT` |
| data_ultima_saida | date | `PCEST.DTULTSAIDA` |
| atualizado_em | timestamptz | |

**PK composta:** `(codigo_principal, filial_id)`. `qt_disponivel` é calculado no app: `quantidade - qt_reservada - qt_bloqueada`.

### `sync_logs` — **NOVO 0.2.0**
Histórico de cada sincronização Winthor.

| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| tipo | varchar(20) | `cadastro` / `estoque` |
| filial_id | bigint FK | NULL pra `cadastro` |
| status | varchar(20) | `em_andamento` / `ok` / `falhou` |
| iniciado_em | timestamptz | |
| concluido_em | timestamptz | |
| duracao_ms | int | |
| qtd_produtos / qtd_codigos / qtd_estoque | int | |
| mensagem | varchar(2000) | erros ou avisos (ex: "N truncados", "N descartados") |
| iniciado_por | bigint FK → usuarios | NULL quando vem do agendador |

## Coletas arquivadas — **NOVO 0.3.0**

Toda coleta finalizada no app gera 2 registros (separação header/payload):

### `coletas_arquivadas` (header — listagem rápida)
| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| uuid | uuid UNIQUE | UUID v4 — chave de cruzamento com payload |
| nome | varchar(120) | nome dado pelo operador (ex.: "Márcio (Cabos)") |
| filial_id | bigint FK → filiais | ON DELETE SET NULL |
| dispositivo_id | uuid FK → dispositivos | ON DELETE SET NULL |
| operador_id | bigint FK → usuarios | nullable |
| operador_login | varchar(50) | espelho do login Winthor |
| arquivada_em | timestamptz | server_default `now()` |
| primeira_leitura_em | timestamptz | |
| ultima_leitura_em | timestamptz | |
| qtd_leituras | int | agregado pra evitar contar payload |
| qtd_unicos | int | distinct(codigo_lido) |
| soma_quantidades | numeric(14,3) | |

### `coletas_arquivadas_payload` (payload completo, lazy)
| Campo | Tipo | Notas |
|-------|------|-------|
| uuid | uuid PK FK → coletas_arquivadas.uuid | ON DELETE CASCADE |
| payload | jsonb | toda a coleta (versão_schema, dispositivo, leituras[]) |
| tamanho_bytes | int | reportado na resposta da API |
| criado_em | timestamptz | |

> **Por que separar?** A listagem (filtros + KPIs) carrega só o header e fica rápida; o payload (pode ser MB com 1000+ leituras) só é lido quando o admin abre o detalhe. PostgreSQL TOAST comprime o JSONB automaticamente quando passa de ~2KB.

## Exportação — **NOVO 0.3.0**

### `export_templates`
Layouts salvos pelos admins pra padronizar exportação de coletas (migration `c7d3f4e8a6b2`).

| Campo | Tipo | Notas |
|-------|------|-------|
| id | bigserial PK | |
| nome | varchar(80) | |
| tipo | varchar(10) | `txt` / `excel` / `pdf` |
| publico | bool | se true, todos os usuários veem |
| owner_id | bigint FK → usuarios | ON DELETE SET NULL — preserva templates públicos órfãos |
| config | jsonb | layout completo (ver shapes abaixo) |
| ultimo_uso_em | timestamptz | atualizado em cada `/usar/{uuid}` |
| eh_default | bool | templates default do sistema (não podem ser excluídos) |
| created_at / updated_at | timestamptz | |

**Migration cria 3 templates default públicos** (1 por tipo) pra galera ter algo pra escolher de cara.

**Shapes de config:**

TXT:
```json
{
  "modo": "consolidado",
  "separador": ";",
  "colunas": [{"chave": "codigo", "ordem": 1, "incluir": true}, ...],
  "header": true,
  "prefixo_linha": "",
  "decimal": {"sempre_decimal": false, "separador": ".", "casas": 4},
  "incluir_nao_coletados": false
}
```

Excel (simplificado — colunas fixas):
```json
{
  "modo": "consolidado",
  "decimal": {"sempre_decimal": false, "casas": 3},
  "incluir_nao_coletados": false
}
```

PDF:
```json
{
  "modo": "consolidado",
  "orientacao": "retrato",
  "incluir_logo": true,
  "incluir_resumo": true
}
```

**10 colunas exportáveis** (catálogo fixo em `app/models/export_template.py::COLUNAS_DISPONIVEIS`):
`id_coleta`, `codigo`, `descricao`, `qtd`, `info_extra`, `data_hora`, `codigo_principal`, `operador`, `dispositivo`, `filial`.

## Impressão de etiquetas — **NOVO 0.4.x**

### `impressoras`
Hardware térmico cadastrado (uma linha por impressora física).

| Campo | Tipo |
|-------|------|
| id | serial PK |
| filial_id | FK → filiais (NULL = global) |
| nome | varchar(80) |
| marca | varchar(20) — `argox` / `zebra` / `elgin` / `outra` |
| modelo | varchar(60) |
| endereco_ip | varchar(45) |
| porta | int (default 9100) |
| linguagem | varchar(10) — `zpl` / `epl` / `tspl` |
| largura_mm, altura_mm | numeric(6,2) |
| dpi | int (default 203) |
| ativa | bool |
| online_em | timestamptz (atualizado pelo heartbeat 60s) |
| ultima_falha_em, ultima_falha_msg | timestamptz + text |

UNIQUE `(endereco_ip, porta)` — impede cadastrar 2× o mesmo destino.

### `impressao_layouts`
Templates de etiqueta.

| Campo | Tipo |
|-------|------|
| id | serial PK |
| nome, descricao | string |
| tipo | varchar(30) — `produto` / `gondola` / `promocional` / `custom` |
| largura_mm, altura_mm, dpi | dimensões físicas |
| linguagem_destino | varchar(10) — `zpl` |
| **template_zpl** | text — ZPL Jinja (`{{ campo \| moeda }}`) — o que vai pra impressora |
| **elementos** | jsonb — lista de `{tipo, x, y, fonte, tamanho, campo, ...}` do editor react-konva |
| campos_usados | jsonb — otimização: pre-computa quais campos o layout consome |
| preview_png | bytea — cache do preview Labelary |
| preview_atualizado_em | timestamptz |
| ativo | bool (soft-delete) |

UNIQUE `(nome)`.

### `impressao_jobs`
Auditoria de cada impressão disparada.

| Campo | Tipo |
|-------|------|
| id | serial PK |
| layout_id | FK → impressao_layouts (RESTRICT) |
| impressora_id | FK → impressoras (SET NULL — só_salvar não precisa) |
| filial_id | FK → filiais |
| usuario_login | varchar(50) |
| dispositivo_id | uuid FK → dispositivos |
| **produtos** | jsonb — `[{codigo, copias, dados_resolvidos: {...}}]` |
| copias_total | int |
| status | varchar(20) — `pendente` / `imprimindo` / `concluido` / `falha` / `cancelado` |
| modo | varchar(20) — `imprimir_agora` / `imprimir_depois` / `so_salvar` |
| coleta_avulsa_id, inventario_id | FK opcional pra rastreabilidade de origem |
| tentativas | int |
| erro_msg | text |
| criado_em, enviado_em, concluido_em | timestamptz |

### `impressao_jobs_log`
Eventos por job (debug fino + timeline).

| Campo | Tipo |
|-------|------|
| id | bigserial PK |
| job_id | FK → impressao_jobs (CASCADE) |
| evento | varchar(30) — `criado` / `render_ok` / `render_falha` / `tcp_conectado` / `enviado` / `sucesso` / `falha` / `retry` / `cancelado` |
| detalhes | jsonb (livre) |
| at | timestamptz |

## QA & Config

### `error_logs` — **NOVO 0.2.0** (release 0.1.0 → ajustes em 0.2.0)
Capturado pelo `ErrorLoggingMiddleware` — todo 5xx + 401/403.

| Campo | Tipo |
|-------|------|
| id | bigserial PK |
| ocorrido_em | timestamptz |
| request_id | varchar(40) — também no header `x-request-id` |
| method, path, status_code | strings + int |
| exc_type, mensagem, traceback | text |
| usuario_id | bigint FK |
| dispositivo_id | uuid FK |
| payload | jsonb (headers saneados de Authorization/Cookie + query) |
| user_agent, ip | strings |

### `app_config` — **NOVO 0.2.0**
Chave-valor pra configurações editáveis pela UI sem mexer no `.env`.

| Campo | Tipo |
|-------|------|
| chave | varchar(60) PK |
| valor | text |
| atualizado_em | timestamptz |
| atualizado_por | bigint FK → usuarios |

Chaves em uso (0.3.0): `sync_agendador_ativo`, `sync_cron_hour`, `sync_cron_minute`, `tz_agendador`, `coletas_avulsas_card_campos` (lista JSON dos campos exibidos no card de leitura do app).

## Migrations Alembic

Em `backend/alembic/versions/`. Ordem cronológica:

1. `e65be5e408d9_initial` — 15 tabelas iniciais
2. `dc0d36cb0751_add_error_logs_qa` — error_logs
3. `317184306192_filial_id_nullable_acesso_total_winthor_` — escopo de filial
4. `e73c8cf4123d_estoques_filial_sync_logs` — sync infra
5. `e98e74fd05b8_expandir_produto_e_estoque` — campos enriquecidos do Winthor
6. `3970e821c2b7_alargar_varchar_produto_leitura_estoque` — fix StringDataTruncation
7. `35ced057d3b5_drop_campos_is_collector_add_app_config` — limpa produto + agendador editável
8. `c8ae7b3b1a9b_dispositivo_telemetria` — colunas `versao_app/plataforma/modelo/ultimo_ip/primeiro_pareamento_em/total_leituras`
9. `e30f7e5147c8_coletas_arquivadas_header_payload` — `coletas_arquivadas` + `coletas_arquivadas_payload`
10. `f1b8d9a2c4e5_dispositivo_logs_auditoria` — `dispositivo_logs` + índices
11. `a2c5e7b9d3f1_dispositivo_operador_atual` — `dispositivos.operador_login_atual` + `operador_login_em`
12. `b6e8f0d1a3c9_permissao_area_rotulo` — `permissoes.area` + `permissoes.rotulo` + catálogo expandido (17 chaves)
13. `c7d3f4e8a6b2_export_templates` — `export_templates` + 3 defaults públicos
14. `d9e1f5a8c0b3_permissoes_coletas_arquivadas_granular` — quebra `admin.coletas_arquivadas` em `.ver/.exportar/.excluir` (auto-migra perfis existentes)
