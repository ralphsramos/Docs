# Módulo Inventário — Coletor 2001

FASE 1 (release 0.4.0) — MVP funcional fiel ao [IS Collector](https://iscollector.com/),
com integração nativa ao Winthor (Oracle) + PostgreSQL + Coletor Virtual.

## Conceitos

```
INVENTÁRIO (campanha de contagem por filial + período)
   ├── 1..N SETORES (divisão lógica — ex: "C&C Carpina")
   │      └── 1..N ÁREAS (subdivisão física onde o operador conta — #1, #2…)
   │            └── 1..N LEITURAS (bipes recebidos do app)
   ├── N DISPOSITIVOS (M2M — quais coletores participam)
   └── N PRODUTOS esperados (snapshot da carga no momento)
```

## Estados (state machine)

```
CADASTRADO → EM_ANDAMENTO → FINALIZADO
   (1ª leitura)        (gerenciador encerra)
```

- **CADASTRADO** — criado, sem 1ª leitura ainda. Permite todas as edições.
- **EM_ANDAMENTO** — recebeu ao menos 1 leitura. App pode coletar.
- **FINALIZADO** — encerrado pelo gerenciador. Não aceita mais leituras.

FASE 2+ adiciona `EM_CONFERENCIA`; FASE 4 adiciona `ARQUIVADO`.

## Modal "Novo inventário" — 4 abas (fiel ao IS Collector)

### Aba 1 — Dados gerais
- Filial (dropdown, imutável após criar)
- Nome, Data execução, Anotações
- **Permissão de coleta** (3 modos):
  - `qualquer` — app aceita qualquer código
  - `somente_carregados` — app rejeita códigos fora da carga
  - `carregados_com_confirmacao` — pede confirmação pra desconhecidos

### Aba 2 — Coleta
- **Pallets** (3 modos): `nao`, `simplificado` (+ campo pallets), `detalhado` (+ camadas + caixas)
- **Operadores**: obrigar identificação?
- **Produtividade**: por qtd total / por qtd bipes (FASE 2)

### Aba 3 — Conferência (🚧 FASE 2)

### Aba 4 — Outras opções
- Permitir editar descrição via app
- Permitir só 1 bipe por código (não duplicar)

## Carga de produtos — 3 fontes

| Fonte | Quando usar | Velocidade |
|---|---|---|
| **🗃️ Banco interno** | Dia-a-dia — usa `estoques_filial` já sincado | Rápido |
| **📊 Planilha XLSX/CSV** | Inventário customizado / parcial | Médio (upload) |
| **🔌 Winthor direto** | Produtos novos não sincados / saldo "ao vivo" | Lento (query Oracle) |

**Cabeçalhos aceitos na planilha** (case-insensitive):
- `codigo` / `código` / `codprod` (obrigatório)
- `descricao` / `descrição` / `nome`
- `embalagem` / `emb`
- `unidade` / `und` / `um`
- `qtd` / `quantidade` / `saldo` / `estoque`

**Estratégias:**
- `substituir` — apaga tudo e re-importa (default)
- `mesclar` — upsert (atualiza existentes + adiciona novos)

## Estrutura de banco

8 tabelas em Postgres:

```
inventarios
  ├ id, filial_id, nome, status, configs (JSONB), anotacoes
  ├ origem_dados, data_execucao
  ├ iniciado_em, finalizado_em, ultima_transmissao_em
  ├ qtd_leituras, qtd_unicos, progresso_pct  (KPIs denormalizados)
  └ created_by, created_by_login

setores (id, inventario_id, codigo, nome, ordem)

areas (id, setor_id, numero_area, nome, endereco,
       qtd_bipes, finalizada_em, finalizada_por_login)

inventario_dispositivos (M2M inventario_id ↔ dispositivo_id)
inventario_operadores   (M2M inventario_id ↔ usuario_id)

inventario_produtos     (snapshot da carga)
  ├ PK (inventario_id, codigo_principal)
  ├ descricao, embalagem, unidade_medida
  └ qtd_esperada, qtd_contada (atualizado pelo sync de leituras)

inventario_logs (auditoria por evento — 12 tipos)

leituras (já existia — usada também pra coleta avulsa)
  ├ inventario_id, area_id, dispositivo_id
  ├ codigo_lido, codigo_principal, quantidade
  ├ pallets, camadas, caixas (FASE 1 inventário)
  └ descricao_resolvida, operador_login (snapshot)
```

## API REST — 29 endpoints

### Admin/web (`/api/v1/inventarios/*`) — 25 endpoints

| Método | Path | Permissão |
|---|---|---|
| POST | `/inventarios` | criar |
| GET | `/inventarios` | ver |
| GET | `/inventarios/{id}` | ver |
| PATCH | `/inventarios/{id}` | criar+gerenciar |
| DELETE | `/inventarios/{id}` | criar+gerenciar |
| POST | `/inventarios/{id}/iniciar` | criar+gerenciar |
| POST | `/inventarios/{id}/finalizar` | criar+gerenciar |
| GET | `/inventarios/{id}/logs` | ver |
| GET/POST/PATCH/DELETE | `/inventarios/{id}/setores[/{sid}]` | criar+gerenciar |
| GET/POST/PATCH/DELETE | `…/setores/{sid}/areas[/{aid}]` | criar+gerenciar |
| GET/PUT | `…/dispositivos` | criar+gerenciar |
| GET/DELETE | `…/produtos` | ver / criar |
| POST | `…/produtos/manual` | criar |
| POST | `…/produtos/banco-interno` | criar |
| POST | `…/produtos/winthor` | criar |
| POST | `…/produtos/planilha` | criar |
| GET | `…/relatorio?formato=xlsx\|pdf\|csv` | exportar |

### Device/app (`/api/v1/device/inventarios/*`) — 4 endpoints

| Método | Path | O que faz |
|---|---|---|
| GET | `/device/inventarios` | Lista inventários disponíveis pro device |
| GET | `/device/inventarios/{id}/carga` | Baixa setores + áreas + produtos + configs (offline) |
| POST | `/device/inventarios/{id}/leituras` | Batch upload (sync ao finalizar coleta) |
| POST | `/device/inventarios/{id}/areas/{aid}/finalizar` | Marca área concluída |

## Permissões (5 novas)

| Chave | Área | Default |
|---|---|---|
| `inventarios.ver` | web | admin, supervisor, operador |
| `inventarios.criar` | web | admin, supervisor, operador (operador faz CRUD!) |
| `inventarios.gerenciar` | web | admin, supervisor (edita qualquer inventário) |
| `inventarios.exportar` | web | admin, supervisor |
| `mobile.inventarios.coletar` | mobile | admin, supervisor, operador |

**Regra-chave:** o operador tem `criar` mas não `gerenciar`. Logo:
- Pode criar inventário próprio
- Pode editar/excluir/iniciar/finalizar **somente os que ele criou** (`created_by = user.id`)
- Não pode mexer em inventário de outros operadores

## Frontend (admin web)

### `/inventarios` — Grid
- Filtros: filial · status · data execução · busca · "apenas meus"
- 4 KPIs (Total / Cadastrados / Em andamento / Finalizados)
- Tabela com barra de progresso + ação "Gerenciar"
- Botão "**+ NOVO INVENTÁRIO**" abre modal de 4 abas

### `/inventarios/:id` — Detalhe (réplica do IS Collector)

```
┌─ Card esquerdo ─────────┬─ Área direita ──────────────────┐
│ Nome + status badge     │ Tabs: Setores/áreas · Produtos  │
│ Configs ativas (checks) │       Auditoria                 │
│ Datas (iniciado/fim/    │                                 │
│        última transm)   │ Cards de SETOR                  │
│                         │  └ Balões de ÁREA (verde=100%)  │
│ [📍ADD SETOR]           │                                 │
│ [📦PRODUTOS]            │ Tabela Produtos:                │
│ [📱DISPOSITIVOS]        │  Cod · Desc · Esp · Cont · Div  │
│ [🗃️ESTOQUE][⋮OPÇÕES]    │                                 │
│ Barra progresso         │ Logs: eventos JSONB             │
│ KPIs 3x2                │                                 │
└─────────────────────────┴─────────────────────────────────┘
```

**Menu OPÇÕES (⋮):**
- ✏️ Editar dados (só CADASTRADO)
- ▶️ Iniciar contagem
- 🏁 Finalizar
- 📋 Ver auditoria
- 📊 Baixar Excel · 📄 PDF · 📃 CSV
- 🗑️ Excluir (só CADASTRADO + sem leituras)

## Coletor Virtual (simulador) — fluxo operador

```
┌─ MENU PRINCIPAL ─┐    ┌─ INVENTÁRIOS ─┐    ┌─ EXATRON ─────┐    ┌─ C&C · #2 ────┐
│ INVENTÁRIO →─────┼──→ │ EXATRON       │ →  │ 📍 C&C        │ →  │ Bipar: [___]  │
│ COLETAS AVULSAS  │    │ TESTE 1       │    │  [#1][#2][#3] │    │ Qtd:   [1]    │
│                  │    │ INVENTÁRIO 5  │    │ [📡 TRANSMITIR│    │ - 7891 (5x)   │
│ SINCRONIZAR      │    │               │    │     3 PEND.]  │    │ - 7892 (2x)   │
└──────────────────┘    └───────────────┘    └───────────────┘    │ [🏁 FINALIZAR]│
                                                                   └───────────────┘
```

**Comportamento offline-first:**
- Carga baixa **uma vez** quando o operador abre o inventário
- Leituras acumulam **local** (state React) durante a contagem
- Transmissão **batch** ao clicar "TRANSMITIR" ou "FINALIZAR ÁREA"
- Idempotente por `(dispositivo_id, uuid_local)` — re-envio não duplica

**Regras de coleta (vindas da config):**
- `permissao_coleta=somente_carregados` → app bloqueia código fora da carga
- `permissao_coleta=carregados_com_confirmacao` → mostra alerta, operador decide
- `um_bipe_por_codigo=true` → bloqueia duplicata na mesma área

## Relatórios

`GET /inventarios/{id}/relatorio?formato={xlsx|pdf|csv}` gera:

- **Header** — nome, filial, status, datas
- **KPIs** — produtos · com divergência · esperado · contado · divergência · acuracidade
- **Tabela** — código · descrição · embalagem · UND · esperado · contado · divergência · % acur

PDF sai em paisagem A4. CSV é UTF-8 com BOM (Excel abre certo).

## O que NÃO entra na FASE 1

| Cortado | Vira |
|---|---|
| Conferência de áreas (recontagem) | **FASE 2** |
| Conferência de itens divergentes (BETA) | **FASE 3** |
| Geração de ajuste em PCMOV no Winthor | **FASE 4** |
| Impressão de etiquetas de áreas | **FASE 4** |
| App Android nativo | continua usando Coletor Virtual |

## Como testar localmente

1. Subir backend + frontend pelo PyCharm (Run Config "Coletor 2001 (full)")
2. Abrir `http://localhost:5173/inventarios` (ou via IP da rede)
3. **+ NOVO INVENTÁRIO** → modal 4 abas → criar
4. Na página de detalhe:
   - **📍 ADD SETOR** → "C&C" com 3 áreas (#1, #2, #3)
   - **📦 PRODUTOS** → aba "Banco interno" → "só com estoque positivo" → CARREGAR
   - **📱 DISPOSITIVOS** → marcar coletor virtual da filial → SALVAR
5. Abrir `/coletor-virtual` → pareou + login operador → MENU → INVENTÁRIO
6. Selecionar o inventário → escolher área #1 → bipar códigos
7. **TRANSMITIR** ou **FINALIZAR ÁREA**
8. Volta no admin → vê progresso atualizado + relatório

## Arquivos relevantes

```
backend/
  app/
    models/inventario.py                       # 8 tabelas + state machine
    schemas/inventario.py                      # 24 schemas Pydantic
    api/v1/inventarios.py                      # 25 endpoints admin
    api/v1/device_inventario.py                # 4 endpoints device
    services/inventario_carga_service.py       # 4 fontes de carga
    services/inventario_export_service.py      # XLSX/PDF/CSV
  alembic/versions/
    872a399bdfa4_inventario_fase1_*.py         # migration estrutural
    a8c4e6f2b9d1_permissoes_inventario_*.py    # 5 permissões + atribuição

web/src/
  api/inventarios.ts                           # client tipado
  pages/InventariosPage.tsx                    # grid + filtros + Novo
  pages/InventarioDetalhePage.tsx              # detalhe (5 botões + tabs)
  components/inventario/
    ModalNovoInventario.tsx                    # modal 4 abas
    ModalCargaProdutos.tsx                     # modal 3 fontes
    ModaisGestao.tsx                           # AddSetor + Dispositivos
    ScreenInventarioFluxo.tsx                  # fluxo no Coletor Virtual
```
