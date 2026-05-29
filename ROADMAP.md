# Roadmap — Coletor 2001

Datas são alvos, não compromissos. Cada release fecha um conjunto coerente de funcionalidades e tem tag no Git.

Legenda: ✅ entregue · 🔄 em curso · ⏳ planejado

---

## 0.1.0 — Fundação ✅

- ✅ Estrutura de pastas (backend/web/mobile/docs)
- ✅ ADR-0001 (escolha de stack)
- ✅ Backend FastAPI + `/health`
- ✅ PostgreSQL via SQLAlchemy async + Alembic
- ✅ Oracle via `oracledb` thin
- ✅ Autenticação contra Winthor
- ✅ Modelos Usuario / Perfil / Permissao / Filial / Dispositivo
- ✅ Painel web com login + layout (sidebar/topbar) inspirado nos mockups

## 0.2.0 — Cadastros, sync e QA ✅

### Autenticação & autorização
- ✅ Login real via procedure `PROC_AUTENTICACAO` (descoberta: senha criptografada em `PCEMPR`, NÃO em `PCUSUARI`)
- ✅ Bloqueio de inativos (`SITUACAO='A' AND DTDEMISSAO IS NULL`) + de desativados internamente
- ✅ `/auth/me` com perfis + permissões consolidadas
- ✅ Dependency `require_any_permission` aplicada nas rotas funcionais
- ✅ Frontend `/sem-acesso` quando login passa sem perfil
- ✅ Tabela `app_config` (key/value) com override do `.env`

### Gestão de usuários
- ✅ Tela `/usuarios` com 3 abas: Lista · Vincular · Perfis & permissões
- ✅ CRUD de perfis com checkbox de permissões
- ✅ Importação de empregados do `PCEMPR` (botão "Importar do Winthor")
- ✅ Ativar/desativar com proteção (não pode desativar a si mesmo)
- ✅ **Vincular multi-filial**: checkboxes pra atribuir N filiais de uma vez ou marcar "TODAS as filiais"
- ✅ Sugestão automática da filial do `PCEMPR.CODFILIAL`

### Filiais & escopo
- ✅ Sincronização `PCFILIAL` filtrada por `WINTHOR_FILIAIS_PERMITIDAS` (default `1..8`)
- ✅ `usuarios_perfis` com `filial_id` nullable + `acesso_total` (XOR constraint)
- ✅ Helper `filiais_permitidas(user)` aplicado em coletas-avulsas

### Produtos & estoque
- ✅ Carga `PCPRODUT` enriquecida + `PCEMBALAGEM` (EANs adicionais)
- ✅ Carga `PCEST` enriquecida (QTESTGER + QTRESERV + QTBLOQUEADA + DTULTENT + DTULTSAIDA)
- ✅ Tela `/produtos` com KPIs + tabela por filial + histórico de sync
- ✅ APScheduler integrado no lifespan (cron diário configurável pela UI)
- ✅ Endpoint `GET /produtos/carga/{filial_id}` (gzip, payload único pro Android)

### QA
- ✅ Middleware `ErrorLoggingMiddleware` + tabela `error_logs` + header `x-request-id`
- ✅ Painel `/qa` com 4 cards + tabela filtrável + modal de traceback
- ✅ Testes pytest do login (8/8 verdes)

## 0.3.0 — Dispositivos, Coletor Virtual e Exportação ✅ (release atual)

### Dispositivos (CRUD + auth por token)
- ✅ CRUD em `/dispositivos` com KPIs (total/ativos/online/inativos/nunca pareados)
- ✅ **Token de 9 dígitos numéricos** (facilita teclado físico dos coletores)
- ✅ Autenticação `X-Device-Token` em todos `/api/v1/device/*` (separado de JWT)
- ✅ Telemetria expandida (versão_app, plataforma, modelo, ultimo_ip, ultimo_sync, etc)
- ✅ Rotação de token, ativar/desativar, transferir filial, excluir

### Logs de auditoria do dispositivo
- ✅ Tabela `dispositivo_logs` com 10 tipos de evento padronizados
- ✅ Helper `registrar_evento` chamado automaticamente em todos endpoints
- ✅ Drawer lateral admin "📋 Log" com timeline + auto-refresh
- ✅ Tentativa de login operador com senha errada também logada

### Coleta Avulsa end-to-end
- ✅ App coletor finaliza coleta → `POST /device/coletas/arquivar`
- ✅ Tabelas `coletas_arquivadas` (header) + `coletas_arquivadas_payload` (JSONB)
- ✅ Página admin `/coletas-avulsas` reformulada (inspirada no IS Collector):
  - 6 KPIs reativos aos filtros
  - Filtros: filial, dispositivo multi, operador, data range, busca
  - 4 chips de período rápido (Hoje / 7d / 30d / 90d)
  - Bulk-select com toolbar "Excluir selecionadas"
- ✅ Página dedicada de detalhe `/coletas-avulsas/:uuid`:
  - Card "Dados da coleta" com 10 metadados + lixeira
  - Cards de download (TXT / Excel / PDFs)
  - Tabela de leituras com filtro local

### Coletor Virtual (simulador completo)
- ✅ `/coletor-virtual` simulando app Android dentro do painel
- ✅ Telas: Pareamento → Login Operador → Menu → Coleta Avulsa + Placeholders
- ✅ Design system industrial: Oswald + Inter, primary #006CB5, secondary #EC3237
- ✅ Login do operador (Winthor) — sessão preservada em `dispositivos.operador_login_atual`
- ✅ Menu hambúrguer mostrando operador + dispositivo + token (toggle)
- ✅ Modal "DIGITE SIM" pra confirmar logout (protege contra toques acidentais)
- ✅ Bottom nav com 4 tabs (Conferência/Separação/Inventário/Coletas)
- ✅ Scrollbar visível (azul oficial) + scroll do mouse no celular contido

### Sistema de Exportação (TXT / Excel / PDF)
- ✅ Tabela `export_templates` (JSONB config, público/privado, default)
- ✅ CRUD `/export-templates/` + página admin
- ✅ **Geração TXT** completa (modo, separador, colunas ordenáveis, header, prefixo, decimal)
- ✅ **Geração Excel** com `openpyxl` (cabeçalho azul, descrições em vermelho, zebra cinza, total)
- ✅ **Geração PDF** com `reportlab` (consolidado + linha a linha, layout do mockup IS Collector)
- ✅ Modal `ModalConfigExport` reproduz mockup IS Collector (TXT completo, Excel simplificado)
- ✅ Edição de layout direto na página `/export-templates` (PATCH)
- ✅ Modelos default públicos pra cada tipo

### Permissões granulares Web/Mobile
- ✅ Coluna `area` (web/mobile/comum) + `rotulo` em `permissoes`
- ✅ 17 permissões totais (vs 8 anteriores)
- ✅ `coletas_arquivadas.{ver,exportar,excluir}` separadas (só admin exclui)
- ✅ EditorPerfil com cards por área + "Marcar todas" + chips por área na tabela
- ✅ Hook `usePermissions()` no frontend esconde botões sem permissão

### Preview do app na config de exibição
- ✅ Modal "Configurar exibição do produto no app" com 3 colunas
- ✅ Mini PhoneFrame renderiza card de produto MOCK atualizando em tempo real

### Dev experience
- ✅ 4 Run Configurations PyCharm versionadas (Backend, Frontend, Alembic, Coletor 2001 full)
- ✅ Script `scripts/start_all.bat` (fallback CLI)
- ✅ README atualizado com "Start completo em 1 clique"

## 0.4.0 — App Android (Capacitor) 🔄

Decisão: em vez de reescrever em Kotlin nativo, empacota a SPA web existente
com **Capacitor** (mesmo código do painel/Coletor Virtual roda no APK).

- ✅ Wrapper Capacitor empacota `web/dist` como APK (`mobile/`)
- ✅ Build de APK debug (JDK 21 + Gradle) — `mobile/android/app/build/outputs/apk/debug/`
- ✅ **Tela "Configurar Servidor"** no 1º boot do APK: operador digita o IP do
  backend, testa `/health` e salva (localStorage). URL da API resolvida em
  runtime — funciona em qualquer rede sem rebuild. Botão "Trocar servidor" no menu.
- ✅ `captureInput` ligado pro scanner físico (intent broadcast)
- ⏳ Suporte offline-first no APK (cache de carga + fila de leituras)
- ⏳ Build de APK **assinado** (release) + distribuição interna
- ⏳ Plugin de scanner nativo (`@capacitor-mlkit/barcode-scanning`) integrado às telas

## 0.4.x — Inventário FASE 1 + Auditoria A-Z + Impressão de Etiquetas ✅ (release atual)

### Impressão de Etiquetas ✅
Módulo novo pra impressão de etiquetas térmicas (ARGOX / ZEBRA / ELGIN).
**Detalhes completos:** [`docs/impressao.md`](docs/impressao.md)

- ✅ IMP-1 Backend foundational — 4 tabelas + 14 endpoints + services (TCP raw, Jinja render, Labelary preview)
- ✅ IMP-2 Frontend admin — página `/impressao` com 5 abas:
  - ⚡ Direta (bipa+preview+imprime na hora) · 📋 Coletas (rastreabilidade)
  - 🏷️ Layouts (editor visual react-konva com paleta de campos Winthor)
  - 🖨️ Impressoras (CRUD + 🔌 testar TCP + online status)
  - 📜 Log (auditoria densa + ⬇ download TXT)
- ✅ IMP-3 Mobile — MenuCard "🖨 ETIQUETAS" no Coletor Virtual + worker APScheduler (fila com retry exponencial + heartbeat 60s)
- ✅ 8 permissões (3 base + 5 granulares por aba) com defesa em profundidade
- ✅ Mock printer (`scripts/mock_printer.py`) — TCP server local na :9100 pra testar sem hardware
- ✅ **Impressora via compartilhamento Windows (SMB)** — além do TCP raw, envia ZPL pelo spooler do Windows em modo RAW (`win32print`), pra impressora USB compartilhada na rede (`\\servidor\nome`). Dispatcher TCP/SMB transparente.
- ✅ **Busca por EAN + preço por embalagem** — bipa CODPROD ou código de barras; cada EAN carrega a `QTUNIT` da embalagem (PCEMBALAGEM) e o preço final = unitário × QTUNIT (espelha o BUSCAPRECOS do RTM Winthor)
- ✅ **Layout "Preço Universal"** — 1 template detecta 3 modos automaticamente: NORMAL (preço único), PROMO (DE/POR + ECONOMIZE + VÁLIDO) e À VISTA (tarja). Espelha o RTM #2012 de etiqueta de preço.
- ✅ **Editor visual como fonte-da-verdade** — elementos geram o ZPL ao salvar (reflete na impressão): condição por modo (`{% if %}`), wordwrap + espaço entre linhas, negrito (double-strike), fonte/tamanho/alinhamento, imagem/logo e plano de fundo (PNG→`^GF`), barcode dimensionado
- ✅ **Duplicar layout** — clona pra criar variações sem mexer no original
- ⏳ IMP-4 — Integrações com Coleta Avulsa / Inventário + descoberta de impressoras na rede (mDNS + scan TCP)



### Auditoria A-Z ✅
- ✅ 4 tabelas Postgres (ciclos, compradores, fornecedores, conferências)
- ✅ 21 endpoints REST (CRUD + 5 KPIs + import + consulta cross-ciclo + gerar coletas)
- ✅ Import em lote de planilhas (script `importar_az_lote.py`)
- ✅ Página `/auditoria` com 6 abas (Visão Geral · Por Comprador · Por Mês · Acumulado · QT Dias · Consulta Histórica)
- ✅ Recharts pra gráficos (cards/timeline/área/heatmap)
- ✅ 7 permissões granulares (1 por aba + ver/gerenciar/mobile)
- ✅ Botão "Gerar coletas no app" — cria inventários auto a partir de fornec selecionados
- ✅ Home renovada — dashboard de gestão com hero, KPIs globais, atalhos por permissão
- ✅ **11 ciclos · 6.195 fornecedores · 2.817 conferências importados** (2024-2026 × Construção/Utilidades/Geral × filiais 1/3/6/Todas)

### Inventário FASE 1 ✅

**Detalhes completos:** [`docs/inventario.md`](docs/inventario.md) · [`docs/auditoria-az.md`](docs/auditoria-az.md)

### FASE 1 — MVP funcional ✅
- ✅ 8 tabelas + state machine (CADASTRADO → EM_ANDAMENTO → FINALIZADO)
- ✅ Modal "Novo inventário" com 4 abas (fiel ao IS Collector)
- ✅ Setores + Áreas + Dispositivos (M2M) + atribuição de operadores
- ✅ Carga de produtos com 3 fontes: banco interno · planilha XLSX/CSV · Winthor direto (Oracle)
- ✅ 29 endpoints REST (25 admin + 4 device)
- ✅ Coletor Virtual: fluxo offline-first (lista → setor/área → bipar → transmitir batch)
- ✅ Permissão de coleta (qualquer/só carregados/com confirmação)
- ✅ Pallets simplificado e detalhado (camadas/caixas)
- ✅ Relatórios XLSX/PDF/CSV com KPIs + esperado vs contado + divergência
- ✅ 5 permissões granulares — **operador pode criar inventário próprio** (CRUD restrito ao criador)
- ✅ Auditoria via `inventario_logs` (12 tipos de evento)

### FASE 2 — Conferência de áreas ⏳
- ⏳ Recontagem de áreas com divergência (status PRECISA_CONFERIR)
- ⏳ Auto-criar conferência por área quando recebe coleta
- ⏳ Finalização: manual / 1ª coleta 100% / qualquer coleta 100%
- ⏳ Estado EM_CONFERENCIA no inventário

### FASE 3 — Conferência de itens divergentes (BETA NEW) ⏳
- ⏳ Recontagem item-a-item (não área inteira)
- ⏳ Regra "ignorar não coletados" vs "considerar todos"
- ⏳ Tela mobile dedicada à conferência

### FASE 4 — Integração saída Winthor + impressão ⏳
- ⏳ Geração de ajuste de estoque no Winthor (PCMOV via procedure homologada)
- ⏳ Status ARQUIVADO após sync com Winthor
- ⏳ Impressão de etiquetas de áreas (QR/barcode)
- ⏳ Produtividade em tempo real (itens/h por operador)

## 0.6.0 — Módulo Conferência ⏳

- ⏳ Carga on-demand de pedido (`PCPEDC`/`PCPEDI`) ou XML de NF (`PCNFENT`)
- ⏳ Bloqueio/alerta de código fora da lista
- ⏳ Acurácia quanti/qualitativa
- ⏳ Validação cega no app (alerta se excedeu quantidade)

## 0.7.0 — Módulo Separação ⏳

- ⏳ Modelo de Separação (similar à Conferência, fluxo de saída via `PCNFSAID`)
- ⏳ Toggle de origem (Oracle direto vs API Winthor) por filial
- ⏳ App: tela de separação com leitura sequencial

## 0.8.0 — Regras avançadas de produto ⏳

- ⏳ Coleta de Lote/Série (parâmetro 0/1/2 — re-introduzir `exige_info_extra`)
- ⏳ Contagem por pallet (re-introduzir `unidades_por_pallet`)
- ⏳ Códigos relacionados UI (N EANs → 1 código principal — já modelado)
- ⏳ Excel/PDF: opção "incluir produtos não coletados" (hoje salva no template mas backend ignora)
- ⏳ Endpoint `POST /export-templates/preview` que não persiste (hoje cria template "_rascunho_" temporário)

## 0.9.0 — Operação e qualidade ⏳

- ⏳ Dashboards no painel (visão filial / visão geral)
- ⏳ Logo PNG real da empresa nos PDFs (hoje é texto)
- ⏳ Comparar coletas selecionadas (botão `/compare/single` do IS Collector)
- ⏳ Agrupar coletas selecionadas (botão `/mergedetail`)
- ⏳ Monitoramento Winthor
- ⏳ Hardening: rate limit, CORS afinado, refresh token, MFA opcional

## 1.0.0 — Produção ⏳

- ⏳ Documentação operacional (deploy, backup, rollback)
- ⏳ Treinamento operadores + manual interno
- ⏳ Go-live com 1 filial piloto
