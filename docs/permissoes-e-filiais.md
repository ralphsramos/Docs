# Permissões & Filiais — Coletor 2001

Como o sistema decide o que cada usuário vê e pode fazer.

---

## 1. As três camadas de controle

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Autenticação (Winthor)                                       │
│    `PROC_AUTENTICACAO` valida senha contra PCEMPR.    │
│    Empregado precisa estar ativo (SITUACAO='A' e DTDEMISSAO IS  │
│    NULL) → login OK.                                            │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. Conta ativa no Coletor 2001                                  │
│    Existe registro em `usuarios` (Postgres) com `ativo=true`?   │
│    - Inativo → login bloqueado mesmo se Winthor aceitar.        │
│    - Não existe → cria automaticamente no primeiro login (mas   │
│      sem perfil, então cai em /sem-acesso).                     │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Perfil + filial(is)                                          │
│    Atribuições em `usuarios_perfis (usuario_id, perfil_id,      │
│    filial_id, acesso_total)` decidem:                           │
│      - O QUE pode fazer (permissões do perfil)                  │
│      - ONDE pode fazer (1 filial, várias OU todas)              │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Filiais — o que são, pra que servem

A tabela `filiais` no Postgres é **espelho do `PCFILIAL`** do Winthor.

- **Origem:** `PCFILIAL.CODIGO IN (1..8)` por padrão (controlado pela env `WINTHOR_FILIAIS_PERMITIDAS`).
- **Sincronização:** botão **Sincronizar Winthor** na tela `/filiais` chama `POST /api/v1/filiais/sincronizar` que faz upsert no Postgres.
- **PCFILIAL nessa instalação:** sem coluna `BLOQUEIO` — controle 100% via env. Códigos existentes: `1..8` operacionais + `20, 21, 80, 90, 99` auxiliares (filtrados).

### Pra que servem no sistema?

**Escopo de visibilidade** dos módulos operacionais:

| Módulo | Filtrado por filial? |
|--------|----------------------|
| Coletas avulsas | ✅ aplicado |
| Inventário | ✅ planejado 0.4.0 |
| Conferência | ✅ planejado 0.5.0 |
| Separação | ✅ planejado 0.6.0 |
| Produtos (cadastro) | ❌ (produto é global) |
| Saldo de estoque (`estoques_filial`) | ✅ por filial |
| Usuários, Perfis, Filiais | ❌ (admin com `admin.usuarios` vê tudo) |

## 3. Atribuição usuário → (perfil, filial)

Tabela `usuarios_perfis`:

| coluna | tipo | semântica |
|--------|------|-----------|
| `usuario_id` | FK | quem |
| `perfil_id` | FK | o que pode fazer (permissões consolidadas) |
| `filial_id` | FK nullable | onde — uma filial específica |
| `acesso_total` | bool | se true, ignora `filial_id` e vê **todas** |

**Constraint XOR (`chk_acesso_total_xor_filial`):** garante consistência.

```sql
(acesso_total = TRUE  AND filial_id IS NULL)
OR
(acesso_total = FALSE AND filial_id IS NOT NULL)
```

### Três modelos de atribuição

| Cenário | Como modelar | Linhas em `usuarios_perfis` |
|---------|--------------|------------------------------|
| **Operador fixo numa filial** | 1 atribuição `(perfil, filial_id=X, acesso_total=false)` | 1 |
| **Supervisor regional em N filiais** | N atribuições `(perfil, filial_id=X1..Xn, acesso_total=false)` | N |
| **Admin global** | 1 atribuição `(perfil, filial_id=NULL, acesso_total=true)` | 1 |

> A aba **Vincular perfil** do painel suporta os 3 casos: grid de checkboxes pra marcar 1 ou N filiais OU o switch "Acesso a TODAS" pra o caso global.

### Por que `acesso_total` em vez de "uma linha por filial"?

Quando uma filial nova for criada depois (loja inaugurada), o usuário com `acesso_total=true` automaticamente ganha acesso. Se fosse uma linha por filial, esquecer de adicionar seria comum.

## 4. Convenção do filtro

A função `filiais_permitidas(user)` em [`backend/app/api/deps.py`](../backend/app/api/deps.py):

| Caso | Retorno |
|------|---------|
| Usuário tem `admin.usuarios` em qualquer atribuição | `None` (vê tudo) |
| Usuário tem qualquer atribuição com `acesso_total=true` | `None` (vê tudo) |
| Usuário tem atribuições só em filiais específicas | `[1, 3, 5]` (lista) |
| Usuário sem nenhuma atribuição | `[]` (não vê nada — já era barrado pelo `require_any_permission`) |

Cada endpoint operacional usa esse resultado pra aplicar `WHERE filial_id IN (...)`.

## 5. Sugestão automática de filial (PCEMPR.CODFILIAL)

Quando um empregado é **importado do Winthor** (tela Usuários → "+ Importar do Winthor"), o sistema:

1. Lê `PCEMPR.CODFILIAL` (ex.: `'1'` pra Carpina).
2. Salva em `usuarios.winthor_codfilial`.
3. Na aba **Vincular perfil**, ao escolher esse usuário, a filial é **pré-marcada automaticamente** no grid de checkboxes (admin pode adicionar mais ou marcar "Todas").
4. No próximo login, o sistema atualiza esse campo (se mudou no Winthor).

## 6. Os 3 perfis padrão (criados pelo seed) — atualizado 0.3.0

| Perfil | Permissões | Tipicamente é... |
|--------|-----------|------------------|
| `admin` | **todas** (17, inclusive `coletas_arquivadas.excluir`) | TI, gestor de operação |
| `supervisor` | mobile.* + `qa.ver` + `admin.coletas_arquivadas` + `inventario.efetivar` + `separacao.efetivar` | Supervisor de filial |
| `operador` | apenas `mobile.*` (coleta_avulsa, inventario, conferencia, separacao, arquivar_coleta) | Operador de coletor |

Diferença entre supervisor e operador: o supervisor enxerga os logs de QA (`qa.ver`), pode ver coletas arquivadas no painel e efetivar inventário/separação no Winthor.

## 7. Lista de permissões — 17 chaves (release 0.3.0)

Cada permissão tem uma **área** (`web` / `mobile` / `comum`) que controla em qual seção da UI ela aparece pra ser marcada num perfil.

### 🖥 Web (painel admin — 9 chaves)

| Chave | O que libera |
|-------|--------------|
| `admin.usuarios` | Gerir usuários, perfis, filiais e agendador (admin geral) |
| `admin.dispositivos` | CRUD dispositivos + ver logs de auditoria |
| `admin.filiais` | Gerir filiais |
| `admin.produtos` | Disparar sync de cadastro/estoque, ver KPIs |
| `admin.coletas_arquivadas` | (legado — preferir granulares abaixo) |
| `qa.ver` | Acessar `/qa` (logs de erro, métricas) |
| `coletas_arquivadas.ver` | Listar e ver detalhe das coletas arquivadas |
| `coletas_arquivadas.exportar` | Baixar TXT/Excel/PDF de coletas |
| `coletas_arquivadas.excluir` | **Apagar coletas (restrito ao admin)** |
| `inventario.efetivar` | Efetivar contagem de inventário no Winthor |
| `separacao.efetivar` | Efetivar separação no Winthor |

### 📱 Mobile (app coletor — 9 chaves)

| Chave | O que libera |
|-------|--------------|
| `mobile.coleta_avulsa` | Coleta avulsa no app |
| `mobile.inventario` | Inventário no app |
| `mobile.conferencia` | Conferência no app |
| `mobile.separacao` | Separação no app |
| `mobile.arquivar_coleta` | Finalizar e arquivar coleta |
| `coleta.criar` | Versão legada de `mobile.coleta_avulsa` |
| `inventario.criar` | Versão legada de `mobile.inventario` |
| `conferencia.criar` | Versão legada de `mobile.conferencia` |
| `separacao.criar` | Versão legada de `mobile.separacao` |

### 🖨 Impressão (módulo de etiquetas — granulares por aba)

| Chave | O que libera |
|-------|--------------|
| `impressao.gerenciar` | **Guarda-chuva**: CRUD em tudo (layouts, impressoras, jobs) |
| `impressao.ver` | **Guarda-chuva**: vê todas as abas (read-only) |
| `impressao.imprimir` | Mobile dispara impressão (`POST /device/impressao/imprimir`) |
| `impressao.direta.usar` | Aba ⚡ Direta (admin web) — bipa + imprime na hora |
| `impressao.coletas.ver` | Aba 📋 Coletas (rastreabilidade + drawer + reimprimir) |
| `impressao.layouts.ver` | Aba 🏷️ Layouts (visualizar templates) |
| `impressao.impressoras.ver` | Aba 🖨️ Impressoras (visualizar hardware + testar conexão) |
| `impressao.log.ver` | Aba 📜 Log (auditoria denso + download TXT) |

Aplicação: a página `/impressao` filtra abas pelo `me.permissoes`. Endpoints
backend usam `require_any_permission(<aba específica>, "impressao.ver",
"impressao.gerenciar")` — defesa em profundidade.

### 🔁 Comum
Nenhuma por enquanto — reservado pra funcionalidades que valem nos dois lados (ex.: visualizar histórico).

### Cuidado especial: `coletas_arquivadas.excluir`
A migration `d9e1f5a8c0b3` quebrou a antiga `admin.coletas_arquivadas` em 3 granulares. Quem tinha a antiga ganhou `.ver` + `.exportar` automaticamente. **Só o perfil "admin" recebeu `.excluir`** — supervisores e operadores conseguem ver e baixar coletas mas não apagar.

A UI aplica isso esconde botão 🗑 (linhas individuais), coluna de checkbox de bulk-select e toolbar "Excluir selecionadas" quando o user não tem a permissão (hook `usePermissions` no frontend). **Segurança real é no backend** — endpoints DELETE têm `_PERM_EXCLUIR` aplicado.

## 8. Fluxo do dia a dia

**Cenário: novo operador na filial 3**

1. RH cadastra ele no Winthor (PCEMPR).
2. Admin do Coletor 2001 abre **Usuários → Lista → + Importar do Winthor**, encontra o novo cara, clica **Importar**.
3. Sistema cria registro em `usuarios` com `winthor_codfilial='3'`.
4. Admin vai em **Vincular perfil**, escolhe o usuário (filial 3 pré-marcada), perfil `operador`, clica **Atribuir 1**.
5. Operador faz login pela primeira vez → cai em Home → só vê coletas/inventários da filial 3.

**Cenário: supervisor regional em 4 filiais (3, 5, 6, 7)**

1. Mesmo passo 1-3.
2. Admin atribui perfil `supervisor`, marca filiais 3, 5, 6, 7 no grid de checkboxes, clica **Atribuir 4**.
3. Sistema cria 4 atribuições em sequência; reporta sucessos/falhas individuais.
4. Supervisor faz login → vê coletas dessas 4 filiais.

**Cenário: gerente global**

1. Atribui perfil `supervisor` + marca **Acesso a TODAS as filiais**.
2. Sistema cria 1 atribuição com `filial_id=NULL, acesso_total=true`.
3. Vê tudo, inclusive filiais futuras.

**Cenário: desligamento**

1. Admin abre **Usuários → Lista**, busca o nome, clica **Desativar**.
2. Próximo login dele → 401 (mesmo que Winthor ainda aceite a senha).
3. Atribuições continuam no banco mas inativas — útil pra auditoria histórica.

## 9. Onde mexer pra ajustar regras

| Quer mudar... | Mexer em... |
|---------------|-------------|
| Quais filiais do PCFILIAL aparecem | `.env` → `WINTHOR_FILIAIS_PERMITIDAS` |
| O que cada perfil pode fazer | Painel `/usuarios` → aba **Perfis** |
| Acesso de um usuário | Painel `/usuarios` → aba **Vincular** |
| Lógica do filtro (ex: adicionar perfil "super-admin") | [`backend/app/api/deps.py`](../backend/app/api/deps.py) → `filiais_permitidas()` |
| Aplicar filtro em novo módulo | Igual coletas: receber `visiveis = await filiais_permitidas(session, user.id)` e passar pro service |
| Horário da sync diária | Painel `/produtos` → botão "⋯" no card do agendador (não precisa mexer no `.env`) |
