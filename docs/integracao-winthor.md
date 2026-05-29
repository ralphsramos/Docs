# Integração com Winthor (Oracle)

> **Atenção:** este documento reflete a estrutura real do Winthor da empresa (Oracle 19c, schema `SCHEMA_ERP`). Outras instalações podem variar — partes documentadas como "nessa instalação" precisam ser revalidadas antes de adaptar.

## Conexão

`oracledb` em modo **thin** (não exige Oracle Instant Client):

```python
import oracledb
conn = await oracledb.connect_async(
    user=settings.WINTHOR_USER,
    password=settings.WINTHOR_PASSWORD,
    dsn=settings.WINTHOR_DSN,  # ex: "ora-prd:1521/winthor"
)
```

A conexão Oracle é configurada via `.env` (variável `WINTHOR_CONECT` no formato EZ-Connect) e o `Settings._expand_winthor_conect` faz o parsing automático. Os valores reais (usuário, host, porta, serviço) ficam só no `.env` local — nunca versionados.

Schema padrão da empresa: **`SCHEMA_ERP`** (~50 tabelas `PC*`).

## Autenticação (login compartilhado)

**Não é simples comparação de senha** — o Winthor usa procedure PL/SQL oficial:

```
PROC_AUTENTICACAO(
  pvc2NomeUsuario   IN  VARCHAR2,  -- login digitado
  pvc2SenhaDigitada IN  VARCHAR2,  -- senha em texto plano
  pvc2Mensagem      OUT VARCHAR2,  -- 'OK' ou 'ERRO'
  pvc2SenhaWinthor  OUT VARCHAR2,
  pvc2Matricula     OUT NUMBER
)
```

A procedure valida a credencial internamente no ERP — o backend não lê nem armazena a senha, só recebe o resultado (ok/erro) da `PROC_AUTENTICACAO`.

Após autenticar, o backend:
1. Confirma que o empregado está ativo (`SITUACAO='A' AND DTDEMISSAO IS NULL`) em `PCEMPR`.
2. Espelha o registro em `usuarios` (Postgres) com `winthor_login = USUARIOBD.upper()`.
3. Atualiza `winthor_codfilial` (de `PCEMPR.CODFILIAL`) pra que a UI sugira a filial padrão na atribuição.
4. Se o admin desativou o usuário internamente (`usuarios.ativo=false`), bloqueia o login com 401 mesmo que o Winthor aceite a senha — admin precisa reativar pelo painel.

Implementação: `backend/app/integrations/winthor/oracle_adapter.py::autenticar`.

### Login do operador NO APP (release 0.3.0)

O app coletor reusa o mesmo fluxo:

1. Operador digita login + senha Winthor na tela `ScreenLoginOperador`.
2. App chama `POST /api/v1/device/login-operador` com `X-Device-Token` no header.
3. Backend chama `autenticar_e_sincronizar` (mesma função do `/auth/login` do painel) — `PROC_AUTENTICACAO` + sync da tabela `usuarios` local.
4. Em vez de emitir JWT, salva o login em `dispositivos.operador_login_atual` + `operador_login_em`.
5. Daí pra frente, todas as leituras enviadas pelo app são automaticamente atribuídas a esse operador (sem o app precisar passar nada extra).
6. `POST /api/v1/device/logout-operador` limpa o campo (dispositivo continua pareado).

Tentativas com senha errada gravam `login_operador_falhou` no `dispositivo_logs` (sem expor a senha) pra auditoria de tentativas suspeitas.

## Cargas

### Cadastro de produtos (`sincronizar_cadastro`)

**Origem:** `PCPRODUT` + `PCEMBALAGEM` (LISTAGG DISTINCT dos CODAUXILIAR extras).

**Query (simplificada):**
```sql
SELECT p.CODPROD, p.DESCRICAO, p.CODFAB, p.CODAUXILIAR,
       p.UNIDADE, p.UNIDADEMASTER, p.CODNCMEX, p.MULTIPLO,
       LISTAGG(DISTINCT e.CODAUXILIAR, ',') WITHIN GROUP (ORDER BY e.CODAUXILIAR) AS EANS_EXTRA
  FROM PCPRODUT p
  LEFT JOIN PCEMBALAGEM e
    ON e.CODPROD = p.CODPROD
   AND e.CODAUXILIAR IS NOT NULL
   AND TRIM(e.CODAUXILIAR) <> NVL(TRIM(p.CODAUXILIAR), '~~~')
 GROUP BY p.CODPROD, p.DESCRICAO, p.CODFAB, p.CODAUXILIAR,
          p.UNIDADE, p.UNIDADEMASTER, p.CODNCMEX, p.MULTIPLO
```

**Destino:**
- `produtos.codigo_principal` ← `CODPROD`
- `produtos.descricao` ← `DESCRICAO`
- `produtos.codigo_fabricante` ← `CODFAB`
- `produtos.codigo_barras_principal` ← `CODAUXILIAR`
- `produtos.unidade` ← `UNIDADE`
- `produtos.unidade_master` ← `UNIDADEMASTER`
- `produtos.codigo_ncm` ← `CODNCMEX`
- `produtos.multiplo` ← `MULTIPLO`
- `codigos_relacionados.codigo_barras` ← `EANS_EXTRA` explodido

**Estratégia:**
- Upsert em **batches de 1000** (escala pros ~234k produtos da empresa).
- **Dedup defensivo** em Python (CODPROD único, EAN único) — previne `CardinalityViolationError` quando o Winthor traz registros conflitantes.
- **Truncate defensivo** de cada campo aos limites das colunas locais. Reporta `N produtos truncados` na mensagem do `SyncLog`.
- Conflitos de EAN (mesmo código apontando pra 2 CODPRODs) também são reportados em vez de quebrar.

### Estoque por filial (`sincronizar_estoque_filial`)

**Origem:** `PCEST` filtrado por filial **E** por saldo não-zero.

**Query:**
```sql
SELECT CODPROD, CODFILIAL,
       NVL(QTESTGER,    0) AS QTESTGER,
       NVL(QTRESERV,    0) AS QTRESERV,
       NVL(QTBLOQUEADA, 0) AS QTBLOQUEADA,
       DTULTENT, DTULTSAIDA
  FROM PCEST
 WHERE CODFILIAL = :filial
   AND ( NVL(QTESTGER,    0) <> 0
      OR NVL(QTRESERV,    0) <> 0
      OR NVL(QTBLOQUEADA, 0) <> 0 )
```

> Produtos com TUDO zero não vão pro Postgres — economiza ordens de magnitude (de 234k → ~30k por filial).

**Destino:** `estoques_filial` (PK `codigo_principal + filial_id`). O job faz `DELETE WHERE filial_id=:f` + `INSERT` em batches — snapshot atômico por filial.

**Filtro extra:** só importa saldos de produtos que já existem em `produtos` localmente (reporta `N descartados` quando você esquece de rodar o cadastro antes).

### Filiais (`sincronizar_filiais`)

**Origem:** `PCFILIAL`. **Sem coluna `BLOQUEIO`** nessa instalação — controle de quais filiais aparecem fica 100% por env.

**Query:**
```sql
SELECT CODIGO, NVL(FANTASIA, RAZAOSOCIAL) AS nome
  FROM PCFILIAL
 WHERE CODIGO IN ('1','2','3','4','5','6','7','8')   -- vem de WINTHOR_FILIAIS_PERMITIDAS
 ORDER BY CODIGO
```

Códigos existentes no PCFILIAL (2026-05-25): `1, 2, 3, 4, 5, 6, 7, 8, 20, 21, 80, 90, 99`. Os `1..8` são operacionais; demais são auxiliares (filtrados via `.env`).

### Empregados (`listar_empregados`)

**Origem:** `PCEMPR` filtrado por ativos.

**Query:**
```sql
SELECT MATRICULA, USUARIOBD, NOME, EMAIL, CODFILIAL, CODUSUR, SITUACAO
  FROM PCEMPR
 WHERE USUARIOBD IS NOT NULL
   AND SITUACAO='A' AND DTDEMISSAO IS NULL
 ORDER BY USUARIOBD
```

Usado pelo botão **"+ Importar do Winthor"** na tela `/usuarios` pra trazer empregados que ainda não foram convidados pro Coletor 2001.

## Carga pro app Android (`GET /produtos/carga/{filial_id}`)

Endpoint dedicado pro app baixar tudo numa chamada. Estrutura:

```json
{
  "filial_id": 1,
  "codigo_filial_winthor": 1,
  "filial_nome": "C&C Carpina",
  "gerado_em": "2026-05-25T15:00:00Z",
  "total_produtos": 31420,
  "total_com_saldo": 28530,
  "ultima_sync_cadastro": "2026-05-25T02:14:00Z",
  "ultima_sync_estoque": "2026-05-25T02:18:30Z",
  "produtos": [
    {
      "codigo_principal": "1001",
      "codigo_fabricante": "FAB001",
      "codigo_barras_principal": "7891234567890",
      "descricao": "Produto X",
      "unidade": "UN", "unidade_master": "CX",
      "codigo_ncm": "22030000",
      "multiplo": "1.000",
      "eans": ["7891234567891", "..."],
      "qt_estoque": "100.000",
      "qt_reservada": "5.000",
      "qt_bloqueada": "2.000",
      "qt_disponivel": "93.000",
      "data_ultima_entrada": "2026-05-20",
      "data_ultima_saida": "2026-05-24"
    }
  ]
}
```

Suporta `?apenas_com_saldo=true` pra reduzir payload. Compressão gzip automática pelo uvicorn — payload típico cai pra ~10x menor.

## Agendamento

**APScheduler** (cron diário, default `02:00 America/Sao_Paulo`) roda `sincronizar_tudo_agendado`:
1. Cadastro global (PCPRODUT + PCEMBALAGEM)
2. Estoque de cada filial sequencialmente

Configuração efetiva sobrepõe `.env` quando há registros em `app_config`:

| Chave em `app_config` | Default `.env` |
|------------------------|----------------|
| `sync_agendador_ativo` | `SYNC_AGENDADOR_ATIVO=true` |
| `sync_cron_hour` | `SYNC_CRON_HOUR=2` |
| `sync_cron_minute` | `SYNC_CRON_MINUTE=0` |
| `tz_agendador` | `TZ_AGENDADOR=America/Sao_Paulo` |

Editado via UI (`⋯` no card "Agendador automático" em `/produtos`) → `PUT /produtos/agendador/config` → `reschedule_job` no APScheduler **sem restart**.

## Devolução de ajustes (planejado 0.4.0)

Ao "Efetivar no Winthor" um inventário ou separação:

1. Backend gera payload `(codprod, codfilial, qt_ajuste, motivo)`.
2. Chama procedure homologada — **a definir com TI** (provavelmente `PKG_AJUSTE_ESTOQUE.GRAVAR` ou similar).
3. Resultado registrado em `efetivacoes_winthor` (a criar) pra auditoria.

Atualmente `OracleAdapter.gravar_ajuste_estoque` é `NotImplementedError`.

## Tabelas PC* relevantes

| Tabela | Uso atual | Release |
|--------|-----------|---------|
| `PCEMPR` | Login (`USUARIOBD`) + lista de empregados | 0.2.0 ✅ |
| `PCFILIAL` | Filiais (CODIGO/FANTASIA/RAZAOSOCIAL) | 0.2.0 ✅ |
| `PCPRODUT` | Cadastro de produto | 0.2.0 ✅ |
| `PCEMBALAGEM` | EANs adicionais | 0.2.0 ✅ |
| `PCEST` | Saldo de estoque (QTESTGER/QTRESERV/QTBLOQUEADA + datas) | 0.2.0 ✅ |
| `PCUSUARI` | Não usar pra login nessa instalação. Útil pra cruzar com `PCEMPR.CODUSUR` (cadastro RCA) | — |
| `PCTABPR` (988k) | **Preço de tabela por região** (PTABELA="DE", PVENDA="POR") | 0.8.0 ⏳ |
| `PCPRECOPROM` (82k) | **Preço fixo/promocional** com validade (PRECOFIXO + DTINICIOVIGENCIA/FIM) | 0.8.0 ⏳ |
| `PCPRECO` (6,1M) | Histórico de alterações de preço + ofertas (DTOFERTAINI/FIM) | 0.8.0 ⏳ |
| `PCINVENT*` (10+ tabelas) | Inventário (cabeçalho + itens + endereço WMS + lotes + peças) | 0.5.0 ⏳ |
| `PCBONUSICONF` (146k) | Conferência de bonificação | 0.6.0 ⏳ |
| `PCCONFERENCIAWMS` / `PCITEMLOTECONFERIDO` | Conferência WMS | 0.6.0 ⏳ |
| `PCMOVEND` / `PCMOVENDPEND` | Separação (movimentos pendentes) | 0.7.0 ⏳ |
| `PCPEDC` / `PCPEDI` | Conferência de pedido | 0.6.0 ⏳ |
| `PCNFENT` | Conferência via NF de entrada | 0.6.0 ⏳ |
| `PCNFSAID` | Separação via NF de saída | 0.7.0 ⏳ |

## Procedures Winthor encontradas

| Procedure | Módulo | Uso futuro |
|-----------|--------|------------|
| `SCHEMA_ERP.ATUALIZARINVENTARIOWMSSAAS` | Inventário | Efetivar inventário do app no ERP |
| `US4.SP_PROCESSA_INVENTARIO` | Inventário | Encerramento de inventário |
| `US4.SP_PROCESSA_INVENTARIO_ACERTO` | Inventário | Acerto de estoque pós-contagem |
| `US4.SP_PROCESSA_CONTAGEM_SEQ` | Inventário | Contagem sequencial (ciclos) |
| `INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_ERP` | Conferência | Lista itens do pedido pra conferir |
| `INTEGRA.P_OBTEM_CONFERENCIA_PEDIDO_WMS` | Conferência | Mesmo via WMS |
| `US4.SP_PROCESSA_CONF_BONUS` | Conferência | Processa conferência de bonificação |
| `PROC_AUTENTICACAO` | Auth | Login (já em uso) |

> Detalhamento completo de cada fluxo (tabelas, colunas, fluxo no app, query SQL DE/POR
> com validação empírica) em [`winthor-fluxos-operacionais.md`](winthor-fluxos-operacionais.md).

## Scripts de investigação

Em `scripts/`:

```powershell
..\venv\Scripts\python.exe ..\scripts\explorar_winthor.py             # estrutura geral
..\venv\Scripts\python.exe ..\scripts\explorar_winthor_preco.py        # detalhes de preço
..\venv\Scripts\python.exe ..\scripts\explorar_winthor_preco_final.py  # query DE/POR validada
```

Não comitam — só imprimem no terminal pra inspeção. Reusam credenciais do `.env`.
