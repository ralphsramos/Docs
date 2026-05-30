# Winthor — Fluxos operacionais (Conferência, Separação, Inventário, Preços)

> Investigação prática feita no Oracle Winthor da empresa (schema `SCHEMA_ERP`, Oracle 19c).
> Outras instalações Winthor podem ter tabelas com nomes parecidos mas colunas diferentes — sempre validar antes de adaptar.

## Resumo executivo

| Fluxo | API/Procedure principal | Tabelas-chave |
|-------|-------------------------|---------------|
| **Inventário** | `SCHEMA_ERP.ATUALIZARINVENTARIOWMSSAAS`<br>`US4.SP_PROCESSA_INVENTARIO`<br>`US4.SP_PROCESSA_INVENTARIO_ACERTO` | `PCINVENT`, `PCINVENTITENS`, `PCINVENTROT` (98k linhas), `PCINVENTLOTE`, `PCINVENTENDERECO*`, `PCLOGINVENTARIOWMS` |
| **Conferência** | `INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_ERP`<br>`INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_WMS`<br>`US4.SP_PROCESSA_CONF_BONUS`<br>`US4.SP_PROCESSA_CONTAGEM_SEQ` | `PCBONUSICONF` (146k), `PCCONFERENCIAWMS`, `PCCONFABASTECIMENTOWMS`, `PCITEMLOTECONFERIDO`, `PCPECASCONFERIDAS`, `PCLOGCONFERENCIAWMS` |
| **Separação** | (sem procedure dedicada — fluxo via tabelas) | `PCMOVEND`, `PCMOVENDPEND`, `PCMOVENDLOTE`, `PCMOVENDPENDCONFLOTE`, `PCMODELOSEPARACAO` |
| **Preço DE/POR** | — | `PCTABPR` (preço tabela por região, 988k), `PCPRECOPROM` (preço fixo/promo, 82k), `PCPRECO` (histórico, 6,1M), `PCPRODUT.PVENDA`/`PRECOFIXO` |

---

## 1. Inventário

### Procedures Winthor
- **`SCHEMA_ERP.ATUALIZARINVENTARIOWMSSAAS`** — atualiza inventário do WMS SaaS no ERP. Parâmetros a confirmar com TI.
- **`US4.SP_PROCESSA_INVENTARIO`** — processa o inventário (encerramento).
- **`US4.SP_PROCESSA_INVENTARIO_ACERTO`** — acerto de estoque após contagem.
- **`US4.SP_PROCESSA_CONTAGEM_SEQ`** — processa contagem sequencial (ciclos).

### Tabelas relevantes
| Tabela | Uso |
|--------|-----|
| `PCINVENT` | Cabeçalho do inventário (vazia nessa instalação — uso esporádico) |
| `PCINVENTITENS` | Itens contados por inventário |
| `PCINVENTROT` (98k) | Histórico de inventário rotativo |
| `PCINVENTROT2`, `PCINVENTROT3` | Histórico complementar |
| `PCINVENTLOTE` | Contagem por lote |
| `PCINVENTPECAS` | Contagem por peça (serial) |
| `PCINVENTENDERECO`, `PCINVENTENDERECOI`, `PCINVENTENDERECOIVOLUME` | Inventário por endereço WMS |
| `PCDIVINVENT` | Divergências detectadas |
| `PCATRIBINVENTRF` | Atribuições de inventário pra RF (coletor) |
| `PCSNGPCINVENT` | Inventário com SNGPC (controlados) |
| `PCLOCALINVENTARIO` (10 linhas) | Locais cadastrados |
| `PCPARAMETROINVENT` | Parâmetros gerais |
| `PCFILTROPRODINVENT` | Filtros de produto por inventário |
| `PCINTEGRACAOINVENTWMS`, `PCINT_ENVIO_INVENTARIO` | Integração WMS |
| `PCLOGINVENTARIOWMS`, `PCLOGATUALIZAINVENT` (42k) | Auditoria |

### Recomendação para o Coletor 2001
Pra **release 0.5.0 (Módulo Inventário)** o fluxo nativo do Coletor 2001 fica:

1. **Coleta no app** → grava em `leituras` local (já implementado).
2. **Finalização** → admin "Efetiva no Winthor" disparando `US4.SP_PROCESSA_INVENTARIO_ACERTO` com payload `(codprod, codfilial, qt_contada, codusur)`.
3. **Auditoria local** → log em `efetivacoes_winthor` (tabela a criar) cruza com `PCLOGATUALIZAINVENT` no Oracle pra rastreio fim a fim.

Confirmar assinatura exata da procedure com TI antes de implementar.

---

## 2. Conferência

### Procedures Winthor
- **`INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_ERP`** — devolve dados de conferência do pedido (modo ERP direto).
- **`INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_WMS`** — mesma coisa mas vinda do WMS.
- **`US4.SP_PROCESSA_CONF_BONUS`** — processa conferência de bonificação (brindes).

### Tabelas relevantes
| Tabela | Uso |
|--------|-----|
| `PCBONUSICONF` (146k) | Itens conferidos de bonificação (histórico grande) |
| `PCCONFERENCIAWMS` | Conferência WMS |
| `PCCONFABASTECIMENTOWMS` | Conferência de abastecimento |
| `PCCONFEXECUCAOOSWMS` | Conferência de OS WMS |
| `PCITEMLOTECONFERIDO` | Item conferido com lote |
| `PCITEMPATRIMONIOCONFERIDO` | Conferência de patrimônio |
| `PCITEMPECACONFERIDO` | Conferência por peça (serial) |
| `PCPECASCONFERIDAS` | Idem (variante) |
| `PCNUMEROSERIECONFERENCIA*` | Conferência por número de série |
| `PCDESCONTOCONF`, `PCDESCONTOCONFGRUPO` | Descontos aplicados em conferência |
| `PCLOGCONFERENCIAFRETE` | Log conferência de frete |
| `PCLOGCONFERENCIAWMS` | Log conferência WMS |
| `PCLOGFLUXOCONF` (77) | Log fluxo de conferência |

### Recomendação para o Coletor 2001
Pra **release 0.6.0 (Módulo Conferência)**:

1. App carrega pedido/NF via `INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_ERP(numpedido)` ou ler de `PCPEDC`/`PCPEDI` direto.
2. Operador bipa item por item → matching contra a lista carregada.
3. Alerta na hora se excedeu quantidade ou item fora do pedido.
4. Finalização grava em tabela local + (futuro) sincroniza com `PCBONUSICONF` ou `PCCONFERENCIAWMS` conforme caso.

---

## 3. Separação

### Procedures Winthor
- **Sem procedure dedicada** — fluxo de separação no Winthor é via tabelas + triggers internas. O processo de separação geralmente acompanha o fluxo de saída (faturamento) usando `PCNFSAID` + `PCMOV`.

### Tabelas relevantes
| Tabela | Uso |
|--------|-----|
| `PCMOVEND` (17 linhas) | Movimento de entrega/saída |
| `PCMOVENDLOTE` | Movimento com controle de lote |
| `PCMOVENDPEND` | Movimentos pendentes (a separar) |
| `PCMOVENDPENDCONFLOTE` | Pendentes com conferência de lote |
| `PCMOVENDPENDCONFLOTELOG` | Log da conferência de lote |
| `PCMOVENDPENDLOG`, `PCMOVENDPENDLOGFIM` | Histórico de pendência |
| `PCMOVENDPENDVOLUME` | Volumes pendentes |
| `PCMODELOSEPARACAO` (9 linhas) | Modelos/templates de separação cadastrados |

### Recomendação para o Coletor 2001
Pra **release 0.7.0 (Módulo Separação)**:

1. App lê movimento pendente de `PCMOVENDPEND` pelo número da nota/pedido.
2. Operador separa item por item bipando os códigos.
3. Finalização atualiza `PCMOVEND` ou grava em tabela própria do Coletor pra evitar lock no Winthor (efetivação batch noturna).

---

## 4. Preço de venda DE / POR (com promoção)

### Estrutura de preços nas a empresa

O Winthor não usa preço único — tem **camadas com prioridade**:

```
┌────────────────────────────────────────────────────────────────────┐
│ PRIORIDADE (alta → baixa)                                           │
│                                                                     │
│  1. PCPRECOPROM (preço fixo/promo com validade)                    │
│     PRECOFIXO + DTINICIOVIGENCIA/DTFIMVIGENCIA + filial/região     │
│                                                                     │
│  2. PCPRECO.POFERTA (oferta com DTOFERTAINI/FIM)                   │
│     por (CODPROD, CODFILIAL, NUMREGIAO)                             │
│                                                                     │
│  3. PCTABPR.PVENDA (preço normal por região)                       │
│     por (CODPROD, NUMREGIAO) — 988k linhas                          │
│                                                                     │
│  4. PCPRODUT.PVENDA (fallback global)                              │
└────────────────────────────────────────────────────────────────────┘
```

**Preço "DE" (cheio):** `PCTABPR.PTABELA` da região. Quando falta, `PCPRODUT.PVENDA`.
**Preço "POR" (vigente):** primeiro nível encontrado na hierarquia acima.

### Tabelas-chave

#### `PCTABPR` (~988k linhas) — preço de tabela por região
| Coluna | Uso |
|--------|-----|
| `CODPROD`, `NUMREGIAO` | PK composta |
| `PTABELA` | **Preço cheio "DE"** |
| `PVENDA` | Preço de venda normal "POR" (sem promoção) |
| `POFERTA`, `POFERTATAB` | Preço de oferta legado |
| `PTABELAATAC`, `PVENDAATAC` | Variações pra atacado |
| `PTABELA1..7`, `PVENDA1..7` | Outras tabelas |

#### `PCPRECOPROM` (~82k linhas) — preço fixo/promocional
| Coluna | Uso |
|--------|-----|
| `CODPRECOPROM` | PK |
| `CODPROD`, `NUMREGIAO`, `CODFILIAL` | Escopo |
| **`PRECOFIXO`** | **Preço promocional (valor — não é flag aqui!)** |
| `DTINICIOVIGENCIA`, `DTFIMVIGENCIA` | Validade |
| `CODCLI`, `CODGRUPOCLI`, `CODREDE`, `CODUSUR` | Pode ser restrito a cliente/grupo (NULL = público) |
| `FRENTECX` ('S'/'N') | Vale pra frente de caixa |
| `CODPLPAGMAX` | Plano de pagamento máximo |

> **Atenção:** em `PCPRODUT.PRECOFIXO` é VARCHAR2(1) (flag 'S'/'N') indicando se o produto USA preço fixo. Em `PCPRECOPROM.PRECOFIXO` é NUMBER com o VALOR. Nomes iguais, semânticas diferentes.

> **`CODPLPAGMAX` distingue promoção a prazo × à vista** (confirmado em produção):
> um mesmo produto/região costuma ter **2 linhas** vigentes em `PCPRECOPROM`:
> uma com `CODPLPAGMAX` **nulo** = promoção "a prazo"/normal, e outra com
> `CODPLPAGMAX = 1` = desconto **à vista** (dinheiro/PIX, normalmente menor).
> O módulo de Impressão sincroniza as duas separadas (`preco_promocional` e
> `preco_avista`) pra a etiqueta poder sair como PROMOÇÃO ou À VISTA.
> Pegar só `MIN(PRECOFIXO)` mistura as duas e perde uma das ofertas.

#### `PCPRECO` (~6,1 milhões de linhas) — histórico de alterações
Por (`CODPROD`, `CODFILIAL`, `NUMREGIAO`). Útil pra ofertas com `DTOFERTAINI`/`DTOFERTAFIM` (ativa entre essas datas).

### Query "DE / POR" pronta pra uso

```sql
SELECT
  p.CODPROD,
  p.DESCRICAO,

  -- PREÇO DE (cheio)
  COALESCE(
    (SELECT MIN(tp.PTABELA) FROM PCTABPR tp
      WHERE tp.CODPROD = p.CODPROD
        AND tp.NUMREGIAO = :regiao
        AND NVL(tp.PTABELA, 0) > 0),
    p.PVENDA
  ) AS PRECO_DE,

  -- PREÇO POR (com hierarquia de promoções)
  COALESCE(
    -- 1. PCPRECOPROM ativa, sem amarra de cliente
    (SELECT MIN(pr.PRECOFIXO) FROM PCPRECOPROM pr
      WHERE pr.CODPROD = p.CODPROD
        AND (pr.CODFILIAL = :filial OR pr.CODFILIAL IS NULL)
        AND NVL(pr.DTINICIOVIGENCIA, SYSDATE) <= SYSDATE
        AND NVL(pr.DTFIMVIGENCIA,    SYSDATE) >= SYSDATE
        AND pr.CODCLI       IS NULL
        AND pr.CODGRUPOCLI  IS NULL
        AND pr.PRECOFIXO IS NOT NULL
        AND pr.PRECOFIXO > 0),
    -- 2. PCPRECO.POFERTA ativa
    (SELECT MIN(pc.POFERTA) FROM PCPRECO pc
      WHERE pc.CODPROD = p.CODPROD
        AND pc.CODFILIAL = :filial
        AND pc.DTOFERTAINI <= SYSDATE
        AND pc.DTOFERTAFIM >= SYSDATE
        AND pc.POFERTA IS NOT NULL
        AND pc.POFERTA > 0),
    -- 3. PCTABPR.PVENDA normal da região
    (SELECT MIN(tp.PVENDA) FROM PCTABPR tp
      WHERE tp.CODPROD = p.CODPROD
        AND tp.NUMREGIAO = :regiao
        AND NVL(tp.PVENDA, 0) > 0),
    -- 4. fallback global
    p.PVENDA
  ) AS PRECO_POR

FROM PCPRODUT p
WHERE p.CODPROD = :codprod
```

**`em_promocao = PRECO_POR < PRECO_DE`**

### Validação empírica (2026-05-26)

- `27.468 produtos` com `PCPRECOPROM` ativa hoje (todos públicos — `CODCLI IS NULL`).
- Exemplos:

| Produto | DE | POR | Desconto |
|---------|----|----|----------|
| COLUNA CELITE SAVEIRO INTEIRA CZ PRATA | R$ 109,90 | R$ 49,90 | **55%** |
| PORTA TOALHA CELITE LOUCA COR PAR | R$ 9,90 | R$ 7,90 | 20% |

`PCPRECO.POFERTA` e `PCOFERTAPROGRAMADAC` estão zerados nessa instalação — a operação usa exclusivamente `PCPRECOPROM` pra preço fixo/promo.

### Quando integrar no Coletor 2001

Adicionar 2 colunas no payload `/produtos/carga/{filial}` e na resposta de leitura:

```json
{
  "codigo_principal": "5",
  "descricao": "COLUNA CELITE SAVEIRO INTEIRA CZ PRATA",
  // ... outros campos existentes ...
  "preco_de": "109.90",
  "preco_por": "49.90",
  "em_promocao": true
}
```

App mostra:
- Sem promoção: só `R$ X,XX`
- Com promoção: `~~De R$ 109,90~~ **Por R$ 49,90** (55% off)`

Trabalho estimado: **release 0.8.0 (Regras avançadas de produto)** — junto com Lote/Pallet, faz sentido.

---

## Como reproduzir essa investigação

Scripts em `scripts/`:

```powershell
# Explora estrutura geral (procedures + tabelas + colunas)
..\venv\Scripts\python.exe ..\scripts\explorar_winthor.py

# Aprofunda em preços (DE/POR + amostras)
..\venv\Scripts\python.exe ..\scripts\explorar_winthor_preco.py

# Validação final com query "produção"
..\venv\Scripts\python.exe ..\scripts\explorar_winthor_preco_final.py
```

Não comitam dados — só imprimem na tela. Reusam `WINTHOR_CONECT` do `.env`.
