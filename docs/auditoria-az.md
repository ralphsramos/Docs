# Auditoria de A-Z — Coletor 2001

Módulo de auditoria cíclica de fornecedores, baseado na planilha legacy
"A-Z Fornecedores Utilidades - Nº Ciclo AAAA".

## Domínio

```
CICLO (1º/2º/3º/4º · AAAA · setor)
   ├── 1..N COMPRADORES (responsáveis — ALYSON, ANDRESSA, MARIA W.…)
   │       └── 1..N FORNECEDORES a auditar (com QT produtos, QT vendas, ranking)
   │              └── 1..2 CONFERÊNCIAS (1ª e opcional 2ª se achou problema)
```

**Lógica:**
- Meta: 3 a 4 ciclos por ano (campo `meta_qtd_ciclos_ano`).
- Cada ciclo é **ATIVO** até ser FINALIZADO ou ARQUIVADO.
- Quando vira o ano, o último ciclo é arquivado e um novo começa.
- Um único ciclo ATIVO por setor por vez.

**Status do fornecedor no ciclo:**
- `pendente` — sem auditoria ainda
- `em_andamento` — em conferência
- `finalizado` — auditado sem problema
- `finalizado_com_auditoria` — auditado, gerou 2ª conferência

## Backend (19 endpoints REST + 4 tabelas + 3 permissões)

### Tabelas
| Tabela | Campos principais |
|---|---|
| `auditoria_ciclos` | nome, ano, numero_ciclo, setor, filial_id, status, datas, meta_qtd_ciclos_ano |
| `auditoria_compradores` | ciclo_id, codigo_winthor (CODFUNC), nome |
| `auditoria_fornecedores` | ciclo_id, comprador_id, codigo_winthor (CODFORNEC), nome, qt_produtos, qt_vendas, ranking, status, mes_pendencia, qtd_nao_encontrados |
| `auditoria_conferencias` | fornecedor_id, ordem (1\|2), data_inicio, data_fim, conferente_login, qt_conferida, observacao, link_arquivo |

### Endpoints
| Método | Path | Permissão |
|---|---|---|
| POST | `/auditoria/ciclos` | gerenciar |
| GET | `/auditoria/ciclos` | ver |
| GET/PATCH/DELETE | `/auditoria/ciclos/{id}` | ver/gerenciar |
| POST | `/auditoria/ciclos/{id}/arquivar` | gerenciar |
| POST | `/auditoria/ciclos/{id}/finalizar` | gerenciar |
| POST | `/auditoria/ciclos/{id}/importar` | gerenciar |
| GET | `/auditoria/ciclos/{id}/compradores` | ver |
| GET | `/auditoria/ciclos/{id}/fornecedores` | ver |
| PATCH | `/auditoria/fornecedores/{id}` | gerenciar |
| PUT/DELETE | `/auditoria/fornecedores/{id}/conferencias/{ordem}` | gerenciar |
| GET | `/auditoria/ciclos/{id}/kpis` | ver |
| GET | `/auditoria/ciclos/{id}/kpis/por-comprador` | ver |
| GET | `/auditoria/ciclos/{id}/kpis/por-mes` | ver |
| GET | `/auditoria/ciclos/{id}/kpis/acumulado` | ver |
| GET | `/auditoria/ciclos/{id}/kpis/qt-dias` | ver |
| POST | `/auditoria/ciclos/{id}/gerar-coletas` | gerenciar |

### Permissões granulares (1 por aba)

| Chave | O que libera | Default |
|---|---|---|
| `auditoria.ver` | Tab Visão Geral (grid editável) | admin, supervisor, operador |
| `auditoria.por_comprador` | Tab Por Comprador (cards) | admin, supervisor, operador |
| `auditoria.por_mes` | Tab Por Mês (timeline + barras) | admin, supervisor, operador |
| `auditoria.kpis_avancados` | Tabs Acumulado + QT Dias (SLA) | admin, supervisor |
| `auditoria.consulta_historica` | Tab Consulta cross-ciclo | admin, supervisor |
| `auditoria.gerenciar` | criar ciclo, importar XLSX, editar fornec/conferência, gerar coletas | admin, supervisor |
| `mobile.auditoria.coletar` | (mobile) gerar coletas a partir do A-Z | admin, supervisor |

**Sistema de filtragem**: o frontend esconde automaticamente as tabs que
o usuário não tem permissão de ver. Backend também valida (defesa em profundidade).

## Frontend (`/auditoria`)

### Layout

```
┌──────────────────────────────────────────────────────────────────┐
│ 📋 Auditoria A-Z  [ATIVO]  [Ciclo▾]  [+ Novo] [📥 XLSX] [...]   │
├──────────────────────────────────────────────────────────────────┤
│ Hero KPIs: 532 fornec · 61% pendente · ████░░ 39% completo      │
├──────────────────────────────────────────────────────────────────┤
│ [Visão Geral] [Por Comprador] [Por Mês] [Acumulado] [QT Dias]   │
├──────────────────────────────────────────────────────────────────┤
│ Conteúdo da tab                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 6 Tabs (cada uma com permissão própria)

1. **🎯 Visão Geral** — grid editável Excel-like
   - Filtros: comprador, status, ranking, mês, busca
   - Edição inline do status (dropdown colorido)
   - Click ✏️ → modal de edição das conferências
   - Checkbox por linha pra seleção em massa → **Gerar coletas**

2. **👥 Por Comprador** — cards com mini-gráfico de pizza
   - 1 card por comprador com avatar, % completo, breakdown por status
   - Visualização rápida do progresso individual

3. **📅 Por Mês** — gráfico de barras empilhadas + cards
   - Status por mês em barras stacked (Recharts)
   - Cards individuais por mês com KPIs

4. **📈 Acumulado** — gráfico de área com Recharts
   - Produtos auditados (mensal + acumulado)
   - Fornecedores auditados (mensal + acumulado)
   - Tabela resumo abaixo

5. **⏱️ QT Dias (SLA)** — heatmap comprador × mês
   - Linhas = compradores, colunas = meses
   - Cor da célula: verde (rápido) → vermelho (lento)
   - Mostra média de dias da 1ª conferência

6. **🔎 Consulta Histórica (cross-ciclo)** — busca em TODOS os ciclos
   - Filtros: CODFORNEC · Nome fornecedor · Comprador · Setor · Ano · Filial · Status
   - Grid: 1 linha por (fornecedor × ciclo) — agrega contadores únicos
   - Drawer lateral: clica numa linha → timeline de conferências (1ª + 2ª) com datas e conferente
   - Use cases:
     - "Em quais ciclos o fornecedor TRAMONTINA foi auditado?"
     - "Quais auditorias o comprador ALYSON fez em 2025?"
     - "Histórico de auditorias na Filial 6 do setor Construção"

## Importação da planilha XLSX

A planilha `A-Z Fornecedores Utilidades - Nº Ciclo AAAA.xlsx` tem 5 abas;
importamos a aba **DADOS** automaticamente. As outras (MÊS, GERAL, GRÁFICOS,
QT DIAS) são **calculadas em tempo real** pelos endpoints `/kpis/*` — não
guardamos duplicado.

### Estrutura da aba DADOS (esperada):
- L1-L5: cabeçalho com totais de compradores
- L7: header real:
  - A=CODIGO_COMPRADOR · B=COMPRADOR · C=CODIGO_FORNEC · D=FORNECEDOR
  - E=QT_PRODUTOS · F=QT_VENDAS · G=RANKING · H=STATUS
  - I-N: 1ª conferência (data_ini, data_fim, conferente, qt, obs, link)
  - O-T: 2ª conferência (idem)
  - V=PRODUTOS_NÃO_ENCONTRADOS · W=MÊS_PENDÊNCIA

### Importação idempotente
- Re-execução **atualiza** fornecedores pelo `(ciclo_id, codigo_winthor)`
- Compradores idem
- Conferências por `(fornecedor_id, ordem)`

### Import em lote (todas as planilhas de uma pasta)

Script `scripts/importar_az_lote.py` detecta auto do nome do arquivo:
ano, número do ciclo, filial e setor. Cria 1 ciclo por planilha
(`ano + numero + setor + filial_id` é a chave única).

```bash
python scripts/importar_az_lote.py "/caminho/pasta/com/planilhas/"
```

Estado atual do banco: **11 ciclos · 6.195 fornecedores · 2.817 conferências**
importados do Google Drive (ciclos 2024–2026, setores Construção/Utilidades/Geral,
filiais 1/3/6/Todas).

### Unique constraint do ciclo

Chave composta: `(ano, numero_ciclo, setor, filial_id)`. Implementado via
2 índices parciais (Postgres trata NULL como sempre distinto na unique
default; precisamos do partial pra garantir):
- `WHERE filial_id IS NOT NULL` → unique (ano, num, setor, filial_id)
- `WHERE filial_id IS NULL` → unique (ano, num, setor)

Permite "1º Ciclo 2026 · Construção · F3" e "1º Ciclo 2026 · Construção · F6"
coexistirem, mas bloqueia 2 ciclos "1º Ciclo 2026 · Construção · todas filiais".

## Botão "📱 Gerar coletas no app" (completo)

A partir dos fornecedores selecionados na grid, cria automaticamente N
**Inventários** no módulo Inventário (status CADASTRADO):

**Estratégias:**
- `um_por_fornecedor` (default) — 1 inventário por fornec, mais granular
- `um_por_comprador` — agrupa todos fornec do mesmo comprador

**O que é criado AUTOMATICAMENTE em cada inventário:**
1. Inventário com nome rastreável: `A-Z {ciclo} · {fornec/comprador} · {1ª/2ª} coleta`
2. **2 setores padrão**: `PDV — Loja` (código `PDV`) e `Estoque — Depósito` (código `EST`),
   cada um com 1 área `#1` inicial
3. Dispositivos selecionados atribuídos (M2M)
4. **Carga automática de produtos** via Winthor (Oracle) filtrando por CODFORNEC
   do fornecedor — estratégia `mesclar` acumula produtos quando há múltiplos
   fornecedores no mesmo inventário

### Ordem da coleta (1ª ou 2ª)

| Ordem | Comportamento |
|---|---|
| **1ª — Geral** | Carrega TODOS os produtos do(s) fornecedor(es) com estoque positivo |
| **2ª — Diferença** | Carrega APENAS produtos que **NÃO foram bipados** na 1ª. Requer `inv_primeira_coleta_id` no body |

Backend pra 2ª: faz `SELECT DISTINCT codigo_principal FROM leituras WHERE inventario_id = :inv1`
e remove esses CODPROD dos produtos carregados via `DELETE … WHERE codigo_principal = ANY(:bipados)`.

### Endpoint auxiliar
- `GET /auditoria/ciclos/{id}/inventarios-gerados` — lista inventários criados
  a partir do ciclo (filtra pelo padrão da anotação). Usado pelo dropdown
  "1ª coleta" do modal quando o admin escolhe 2ª.

Operador abre o app coletor → vê "A-Z 1º Ciclo 2026 · ALYSON · 1ª coleta" →
escolhe área #1 do setor PDV → bipa códigos → finaliza → admin gera 2ª coleta
com a diferença.

## Como testar

1. Já tem dados: ciclo "1º Ciclo 2026" foi importado com 532 fornecedores
2. ▶ Run Config Coletor 2001 (full)
3. `http://localhost:5173/auditoria`
4. Navega pelas 5 tabs
5. Na Visão Geral, edita status inline (dropdown colorido)
6. Clica em ✏️ pra editar conferência
7. Marca fornecedores na checkbox → **📱 Gerar coletas** → vira inventários

## Arquivos relevantes

```
backend/
  app/models/auditoria_az.py                    # 4 tabelas
  app/schemas/auditoria_az.py                   # DTOs Pydantic
  app/services/auditoria_az_import.py           # Parser XLSX
  app/api/v1/auditoria_az.py                    # 19 endpoints
  alembic/versions/
    f41093ecb1ef_auditoria_az_*.py              # tabelas
    d7e8b3c1f6a4_permissoes_auditoria_az.py     # permissões

web/src/
  api/auditoria-az.ts                           # client tipado
  pages/AuditoriaPage.tsx                       # página principal
  components/auditoria/
    TabVisaoGeral.tsx                           # grid editável
    TabsViz.tsx                                 # 4 tabs visuais (cards/timeline/area/heatmap)
    Modais.tsx                                  # NovoCiclo + EditarConferência + GerarColetas
```

Dependência nova no frontend: `recharts` (gráficos).
