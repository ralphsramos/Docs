# Impressão de Etiquetas — Módulo Completo

Módulo de impressão de etiquetas térmicas para impressoras **Argox**, **Zebra**,
**Elgin** (e qualquer outra que aceite ZPL via TCP). Cobre o ciclo completo:
cadastro de hardware → desenho de layouts → impressão (web + mobile) →
auditoria.

> Status: **Backend ✓ · Admin web ✓ · Mobile ✓ · Worker fila ✓**
> Layout de preço com 3 modos (NORMAL/PROMOÇÃO/À VISTA), editor visual com
> cores/tarja/camadas e código de barras responsivo. ✓
> Pendente: integrações com Coleta Avulsa / Inventário (IMP-4).

---

## Novidades (2026-06)

A aba **Lista de produtos** (impressão em lote) ganhou:
- **Busca inteligente** — número ≤6 díg = CODPROD exato · ≥7 díg = CODAUXILIAR/EAN exato (principal ou de embalagem) · texto = `ILIKE %palavra%` por palavra (sem falso-positivo de substring).
- **1 linha por embalagem** — produto com 2+ embalagens (UN/CX/FARDO) expande no grid; cada linha imprime com **seu EAN** e **preço = unitário × qtunit**.
- **Preço a imprimir por produto** — clicar na coluna **Tabela / Promoção / À vista** escolhe o que sai naquele produto (sobrepõe o "Preço a imprimir" global).
- **Promoções múltiplas** — produto com 2+ campanhas do mesmo tipo vira um **seletor** (tabela `promocoes_filial`, migration `c1f9a4b7e602`; sincroniza TODAS as campanhas ativas do PCPRECOPROM).

**Imprimir nesta máquina** (navegador): manda o **mesmo PNG do preview** numa página `@page { size: LxA mm; margin: 0 }` com `object-fit: fill` → impressão **1:1** com o preview (driver no tamanho da etiqueta · margens nenhuma · escala 100%). Modo imagem universal (driver Windows) + gerador PPLA bitmap (Argox) pra impressoras não-ZPL.

A aba **Coletas** passou a mostrar **só as coletas salvas pra imprimir depois** (carrega ao **Buscar**; filtros filial/data/usuário) — o histórico de impressão imediata fica no **Log**.

---

## 1. Decisões de arquitetura

| Decisão | Tech | Justificativa |
|---|---|---|
| Linguagem da etiqueta | **ZPL II** | Argox/Elgin têm modo de emulação Zebra (PPLZ/ZB) — 1 ZPL roda em 95% do parque |
| Transporte | **TCP raw porta 9100** | Padrão da indústria, sem driver/CUPS/cloud, suportado por todas as marcas |
| Async I/O | `asyncio.open_connection` | Não bloqueia event loop do FastAPI durante envio TCP |
| Templating | **Jinja2** | Placeholders `{{ descricao }}`, `{{ preco_por \| moeda }}` no ZPL |
| Preview | **Labelary API** | api.labelary.com renderiza ZPL→PNG; serviço de referência da indústria |
| Editor visual | **react-konva** | Canvas drag-drop com paleta de campos Winthor + propriedades |
| Worker fila | **APScheduler IntervalTrigger 30s** | Retry exponencial 1s/5s/30s/2min/10min (5 tentativas) |
| Heartbeat | **TCP ping :9100 a cada 60s** | Atualiza `impressoras.online_em` → UI mostra bolinha verde/cinza |

## 2. Modelo de dados (4 tabelas)

```
impressoras             — hardware térmico cadastrado (IP+porta+filial+marca+linguagem+DPI)
impressao_layouts       — templates (template_zpl + elementos JSON + preview_png cache)
impressao_jobs          — auditoria de cada impressão (produtos JSON + status + tentativas)
impressao_jobs_log      — eventos por job (criado→render_ok→tcp_conectado→enviado→sucesso/falha)
```

Migrations: `c5b8f3a7e2d9` (foundational) + `d4e6a2c8f1b7` (permissões granulares)
+ `f9a2b6c3e801` (estoques_filial preço DE/POR) + `c7e1a9d4f602` (preço
promocional + à vista separados por plano de pagamento).

## 3. Permissões (granulares por aba)

| Chave | Cobre |
|---|---|
| `impressao.gerenciar`         | **Guarda-chuva** — vê tudo + CRUD em tudo |
| `impressao.ver`               | **Guarda-chuva** — vê todas as abas |
| `impressao.imprimir`          | Mobile dispara impressão |
| `impressao.direta.usar`       | Aba ⚡ Direta (admin web) |
| `impressao.lista.ver`         | Aba 📋 Lista de produtos (impressão em lote) |
| `impressao.coletas.ver`       | Aba 📋 Coletas (coletas salvas pra imprimir depois) |
| `impressao.layouts.ver`       | Aba 🏷️ Layouts (visualizar) |
| `impressao.impressoras.ver`   | Aba 🖨️ Impressoras (visualizar) |
| `impressao.log.ver`           | Aba 📜 Log (auditoria) |

Defesa em profundidade: frontend esconde abas sem permissão, backend valida
cada endpoint com `require_any_permission(...)`.

## 4. As 6 abas do `/impressao`

### ⚡ Direta
Bipa código → preview Labelary com dados reais → IMPRIMIR. Engrenagem ⚙ pra
escolher impressora/layout/filial (salva em localStorage por usuário).
Impressora, layout **e filial** são auto-selecionados (sem filial não há preço).
O painel mostra os **3 preços** (Tabela, Promoção, À Vista) e dois botões
**PROMOÇÃO / À VISTA** pra escolher qual sai na etiqueta — o preview e a
impressão refletem a escolha. Quando o produto não tem desconto, sai o layout
NORMAL (preço único). A tag **⚡ Impressão Direta** (ao lado de Cópias), quando
marcada, imprime na hora ao bipar (ENTER) sem clicar IMPRIMIR — fluxo de balcão.

### 📋 Lista de produtos
Impressão em **lote** a partir do catálogo, espelhando a tela "Filtrar Produtos"
da rotina Winthor de emissão de etiquetas de preço (RTM 2012): escolhe a filial →
define os filtros → **🔍 Buscar** → marca os produtos (com nº de cópias por item,
ou "marcar todos da página") → configura impressora/layout/preço no **ícone ⚙**
do rodapé → **🖨 Imprime os selecionados** de uma vez. Cada linha mostra os 3
preços, fornecedor/departamento e as datas de **alteração de preço** e
**fim da promoção**.

> **Configuração de impressão**: no rodapé, o botão **⚙** mostra um resumo da
> config atual (impressora · layout · preço) e abre um modal pra trocar
> impressora, layout e modo de preço. A borda fica vermelha enquanto faltar
> impressora ou layout.

> **Carga sob demanda**: a lista só é carregada **após clicar em Buscar** (ou
> aplicar um chip/filtro) — não despeja o catálogo inteiro ao abrir a aba nem ao
> trocar de filial.

Filtros na própria página:
- **Busca** por código, EAN ou descrição. A busca por **descrição** quebra o
  texto em **palavras**: cada palavra vira um `ILIKE %palavra%` e **todas**
  precisam casar (AND), em qualquer ordem e em qualquer um dos três campos
  (código / EAN / descrição). Ex.: `cadeira vanda` encontra
  `CADEIRA ESCRITÓRIO VANDA`.
- **Situação** (chips): todos · só com estoque · só em promoção · só com à vista.
- **⚙ Filtros** — abre o modal de filtros avançados (abaixo); a engrenagem mostra
  um badge com a quantidade de filtros ativos.
- **🧾 Lista por código** — abre um modal pra colar uma lista de **códigos
  internos e/ou códigos de barras** (um por linha ou separados por vírgula/espaço).
  Resolve via `POST /resolver-codigos` (casa por CODPROD e, se não achar, por EAN
  principal ou de embalagem), mostra encontrados/não-encontrados e entra no
  **modo lista**.
- **Por NF de entrada** — informa o nº da NF e traz todos os itens recebidos
  naquela nota (consulta o Winthor ao vivo via `/nf-entrada`), entrando no
  **modo lista**, pra etiquetar a mercadoria recém-chegada.

> **Modo lista**: ao usar "Lista por código" ou "NF de entrada", o grid passa a
> mostrar **só os produtos informados** (não o catálogo inteiro) — já marcados
> pra impressão. Um badge "📋 Lista por código/NF" aparece no cabeçalho, com um
> botão "✕ Sair da lista" pra voltar ao catálogo. No backend é o parâmetro
> `codigos` (`IN`) do `/produtos-lista`. Buscar / trocar chip / aplicar um filtro
> avançado saem do modo lista.

Modal **⚙ Filtros (rotina 2012)** — paridade com a tela F5-Filtro do WinThor:
- **Classificação**: Fornecedor, Comprador, Departamento, Seção, **Exceto seção**
  e Marca — cada um é um **combobox com busca (por código ou nome) e múltipla
  seleção** (checkbox); as opções vêm de `/opcoes-filtro`. No backend cada filtro
  é um `IN (...)`, e o **Exceto seção** é um `NOT IN` (`excluir_secao`) que remove
  do grid os produtos das seções escolhidas.
- **Tipo & Estoque**: tipo de produto (em linha / fora de linha, via
  `PCPRODUT.DTEXCLUSAO`), estoque (>0 / =0 / <0) e "somente produto principal".
- **Períodos** (início → fim): alteração de preço (`PCTABPR.DTULTATUPVENDA`),
  promoção (`PCPRECOPROM.DTINICIOVIGENCIA`/`DTFIMVIGENCIA`) e última entrada
  (`PCEST.DTULTENT`).

As colunas da tabela mostram, além dos 3 preços, o fornecedor/departamento de
cada linha e as datas de **alteração de preço** e **fim da promoção**.

**Cores por preço**: cada preço tem sua cor (cabeçalho + valor) — **Tabela** azul,
**Promoção** laranja, **À vista** verde; as datas de promoção (início/fim) saem
em laranja, casando com a coluna Promoção.

**Grid ordenável (tipo Excel)**: clique no cabeçalho de qualquer coluna (Código,
Descrição, EAN, Estoque, Tabela, Promoção, À vista, Alterado, Promo início, Promo
fim) pra ordenar; clique de novo pra inverter (▲/▼). A ordenação é feita no
servidor (`ordenar_por`/`ordem`) com vazios sempre no fim e desempate por código.
A **impressão segue a ordem da grid** — os selecionados saem na sequência exibida,
não na ordem em que foram marcados (selecionados fora da página atual vão ao fim).

### 🗂️ Coletas
Grid de jobs com drawer lateral (payload, ZPL renderizado, log de eventos,
botão reimprimir). Filtros: filial, usuário, status, layout, impressora.

### 🏷️ Layouts
CRUD com **editor visual react-konva**: canvas em mm, paleta de campos
Winthor arrastáveis (descrição, preço DE/POR, EAN, código), propriedades à
direita, **preview Labelary ao lado com debounce 500ms**. Aba secundária 💻 ZPL
pra editar o template bruto. **Duplicar layout** (clona pra criar variações).

Recursos do editor visual:
- **Tipografia**: fonte ZPL, tamanho, alinhamento, rotação, negrito, quebra de
  linha (wordwrap) com espaçamento.
- **Cores (P&B na térmica)**: cor do texto + cor de fundo (texto/campo) e cor
  de linha + cor de fundo (caixa). Como a impressora é monocromática, fundo
  escuro vira **tarja preta** (`^GB` sólido) com texto branco (`^FR`); cores
  claras saem como preto/branco. A cor aparece no canvas; o preview Labelary
  mostra como sai impresso.
- **Camadas (z-order)**: botões ⤓ Fundo / ↓ Recuar / ↑ Avançar / ⤒ Frente no
  painel de Propriedades definem quem fica na frente/atrás. A ordem é salva no
  template e respeitada na impressão direta.
- **Código de barras responsivo**: gerado como imagem (`^GF`) no tamanho exato
  da caixa — preenche a largura/altura sem os saltos do `^BC` nativo.
- **Imagem/logo + plano de fundo**: upload PNG/JPG convertido pra `^GF`.
- **Produto de teste**: campo pra bipar um produto real (cód/EAN) + filial +
  modo (auto/promoção/à vista) e ver o preview com dados reais em vez do mock.

### 🖨️ Impressoras
CRUD de hardware. Cada card mostra bolinha verde (online ≤ 5min) ou cinza.
Botão **🔌 Testar** faz TCP ping + ~HS (status Zebra).

### 📜 Log
Tabela densa tipo log: data/dispositivo/usuário/impressora/layout/qtd/status.
Auto-refresh 15s. **Botão ⬇ TXT** em cada linha baixa arquivo de auditoria
TSV com lista de produtos impressos.

## 5. Tela mobile (Coletor Virtual)

MenuCard **🖨 ETIQUETAS** no menu principal (visível com `impressao.imprimir`).

Fluxo: select impressora (filtrada pela filial do device) + select layout +
bipa códigos → 2 botões:
- **🖨️ IMPRIMIR** (modo `imprimir_agora` — TCP síncrono)
- **💾 SALVAR** (modo `imprimir_depois` — worker pega depois)

Fila local em localStorage sobrevive refresh + troca de aba.

## 6. Workers APScheduler

Registrados em `setup_scheduler()` (sempre ativos):

| Job | Trigger | O que faz |
|---|---|---|
| `impressao_fila`       | 30s | Pega `JOB_PENDENTE` com modo `imprimir_depois`, tenta TCP. Backoff exponencial (1s, 5s, 30s, 2min, 10min). Após 5 tentativas → `JOB_FALHA`. |
| `impressao_heartbeat`  | 60s | TCP ping em cada impressora ativa, atualiza `online_em` ou `ultima_falha_em` + `ultima_falha_msg`. |

## 7. Endpoints REST

```
# Impressoras
GET    /impressao/impressoras
POST   /impressao/impressoras
PATCH  /impressao/impressoras/{id}
DELETE /impressao/impressoras/{id}
POST   /impressao/impressoras/{id}/testar      ← TCP ping + ~HS

# Layouts
GET    /impressao/layouts
GET    /impressao/layouts/{id}
POST   /impressao/layouts
PATCH  /impressao/layouts/{id}
DELETE /impressao/layouts/{id}                 ← soft-delete (ativo=false)
GET    /impressao/layouts/{id}/preview.png     ← cacheado em DB
POST   /impressao/layouts/preview              ← live (editor)

# Catálogo (paleta do editor + aba Direta)
GET    /impressao/campos-disponiveis
GET    /impressao/produtos/{codigo}/dados      ← preview aba Direta

# Jobs / imprimir
POST   /impressao/imprimir
GET    /impressao/jobs
GET    /impressao/jobs/{id}
GET    /impressao/jobs/{id}/produtos.txt       ← download TXT auditoria
POST   /impressao/jobs/{id}/reimprimir

# Mobile (X-Device-Token)
GET    /device/impressao/contexto
POST   /device/impressao/imprimir              ← idempotente por uuid_local
```

## 8. Mock printer pra dev/teste

Script `scripts/mock_printer.py`:

```bash
python scripts/mock_printer.py --salvar-em "C:/temp/etiquetas-test"
```

Sobe TCP server local na `:9100` que aceita ZPL, loga o conteúdo, salva como
`.zpl` que pode ser aberto em https://labelary.com/viewer.html. Cadastra como
impressora `IP 127.0.0.1` no admin pra testar todo o pipeline sem hardware.

## 9. Snapshot ZPL

ZPL gerado pelo layout seed "Etiqueta produto 50x30 padrão":

```
^XA
^PW399
^LL239
^FO20,15^A0N,22,22^FD{{ descricao | upper }}^FS
^FO20,45^A0N,18,18^FD{{ codigo_principal }}^FS
^FO130,45^A0N,18,18^FDR$^FS
^FO165,40^A0N,36,36^FD{{ preco_por | moeda }}^FS
^FO20,90^BCN,50,Y,N,N^FD{{ codigo_principal }}^FS
^XZ
```

Após o render Jinja com dados reais:
```
^FO20,15^A0N,22,22^FDREFRIGERANTE COCA 350ML LT^FS
...
^FO165,40^A0N,36,36^FD5,99^FS
^FO20,90^BCN,50,Y,N,N^FD12345^FS
```

## 10. Layout de preço "Preço Universal" (3 modos)

Layout único (seed `scripts/seed_layout_preco.py`, 70×30mm) que detecta o modo
visual pelos dados do produto, espelhando o RTM Winthor de etiqueta de preço:

| Modo | Quando | Mostra |
|---|---|---|
| **NORMAL**    | sem desconto (POR = tabela)        | preço único + EMISSÃO |
| **PROMOÇÃO**  | preço promocional < tabela         | DE/POR + **ECONOMIZE** + VÁLIDO |
| **À VISTA**   | preço à vista < tabela (escolhido) | DE/POR + tarja preta **"À VISTA (DINHEIRO OU PIX)"** + VÁLIDO |

Condições por elemento (`condicao`): `sempre`, `desconto` (promo **ou** à vista
— cobre DE/POR/VÁLIDO), `promo` (só ECONOMIZE), `avista` (só a tarja), `normal`.
Promoção e à vista são **mutuamente exclusivas** (`em_promocao` XOR `tem_avista`)
e só ligam quando há **desconto real** (preço escolhido < tabela).

Origem dos 3 preços no Winthor (sincronizados em `estoques_filial`):

| Campo | Origem | Observação |
|---|---|---|
| `preco_de` (tabela)   | `PCTABPR.PTABELA`                          | preço cheio da região |
| `preco_promocional`   | `PCPRECOPROM.PRECOFIXO` com `CODPLPAGMAX` nulo | promoção "a prazo" |
| `preco_avista`        | `PCPRECOPROM.PRECOFIXO` com `CODPLPAGMAX = 1`  | desconto à vista / PIX |

A query separa as duas promoções por plano de pagamento e filtra a região
(antes pegava `MIN(PRECOFIXO)` e perdia uma das ofertas).

## 11. Pontos abertos (IMP-4)

- Botão "🖨️ Imprimir etiquetas" dentro da Coleta Avulsa / Inventário (modal
  pra escolher layout + impressora + cópias e dispara batch)
- Descoberta de impressoras na rede (mDNS via `zeroconf` + scan TCP da
  subnet) — botão "🔍 Procurar" na aba Impressoras
- Importar ZPL via upload `.zpl`/`.prn` no editor de layouts
- Sincronizar a vigência da promoção (`PCPRECOPROM.DTFIMVIGENCIA`) pro campo
  VÁLIDO mostrar a data real de fim (hoje cai pra data de emissão)
