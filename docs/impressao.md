# Impressão de Etiquetas — Módulo Completo

Módulo de impressão de etiquetas térmicas para impressoras **Argox**, **Zebra**,
**Elgin** (e qualquer outra que aceite ZPL via TCP). Cobre o ciclo completo:
cadastro de hardware → desenho de layouts → impressão (web + mobile) →
auditoria.

> Status: **Backend ✓ · Admin web ✓ · Mobile ✓ · Worker fila ✓**
> Pendente: integrações com Coleta Avulsa / Inventário (IMP-4).

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

Migrations: `c5b8f3a7e2d9` (foundational) + `d4e6a2c8f1b7` (permissões granulares).

## 3. Permissões (granulares por aba)

| Chave | Cobre |
|---|---|
| `impressao.gerenciar`         | **Guarda-chuva** — vê tudo + CRUD em tudo |
| `impressao.ver`               | **Guarda-chuva** — vê todas as abas |
| `impressao.imprimir`          | Mobile dispara impressão |
| `impressao.direta.usar`       | Aba ⚡ Direta (admin web) |
| `impressao.coletas.ver`       | Aba 📋 Coletas (rastreabilidade) |
| `impressao.layouts.ver`       | Aba 🏷️ Layouts (visualizar) |
| `impressao.impressoras.ver`   | Aba 🖨️ Impressoras (visualizar) |
| `impressao.log.ver`           | Aba 📜 Log (auditoria) |

Defesa em profundidade: frontend esconde abas sem permissão, backend valida
cada endpoint com `require_any_permission(...)`.

## 4. As 5 abas do `/impressao`

### ⚡ Direta
Bipa código → preview Labelary com dados reais → IMPRIMIR. Engrenagem ⚙ pra
escolher impressora/layout/filial (salva em localStorage por usuário).

### 📋 Coletas
Grid de jobs com drawer lateral (payload, ZPL renderizado, log de eventos,
botão reimprimir). Filtros: filial, usuário, status, layout, impressora.

### 🏷️ Layouts
CRUD com **editor visual react-konva**: canvas em mm, paleta de campos
Winthor arrastáveis (descrição, preço DE/POR, EAN, código), propriedades à
direita, **preview Labelary ao lado com debounce 500ms**. Aba secundária 💻 ZPL
pra editar o template bruto. Importar via upload `.zpl` (IMP-4).

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

## 10. Pontos abertos (IMP-4)

- Botão "🖨️ Imprimir etiquetas" dentro da Coleta Avulsa / Inventário (modal
  pra escolher layout + impressora + cópias e dispara batch)
- Descoberta de impressoras na rede (mDNS via `zeroconf` + scan TCP da
  subnet) — botão "🔍 Procurar" na aba Impressoras
- Importar ZPL via upload `.zpl`/`.prn` no editor de layouts
- Duplicar layout
