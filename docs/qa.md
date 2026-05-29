# QA — Coletor 2001

## Visão geral

O sistema captura erros automaticamente e expõe ferramentas pra investigar.

| Canal | O que captura | Onde ver |
|-------|---------------|----------|
| **Logs estruturados (stdout)** | Todas as requisições + exceções (JSON via `structlog`) | terminal onde o `uvicorn` roda |
| **Tabela `error_logs`** | HTTP 5xx + 401/403 (request_id, traceback, payload saneado, IP, UA) | painel **QA / Logs** + `GET /api/v1/qa/logs` |
| **Tabela `sync_logs`** | Histórico de cada sync Winthor (tipo, duração, contagens, mensagem) | painel **Listas de produtos** + `GET /api/v1/produtos/sync-logs` |
| **Tabela `dispositivo_logs`** | **0.3.0** — eventos do dispositivo (pareamento, sync, login operador, leituras, arquivar coleta, logout, etc.) | painel **Dispositivos → botão 📋 Log** por linha + `GET /api/v1/dispositivos/{id}/logs` |
| **Testes pytest** | Login, autorização, integrações stub | `pytest -q` no `backend/` + CI |
| **CI GitHub Actions** | Lint + testes a cada push/PR | https://github.com/ralphsramos/Coletor_2001/actions |

## Painel `/qa`

URL: http://localhost:5173/qa — item na sidebar aparece **só pra quem tem permissão `qa.ver`** (padrão: perfis `admin` e `supervisor`).

- **Cards de resumo**: total de erros, últimas 24h, último erro, distribuição por status.
- **Tabela** com filtros por status e busca por path.
- **Clique numa linha** → modal com **traceback completo**, payload (query, headers saneados), request_id, IP, user agent.

## Como rodar os testes

```powershell
cd backend
..\venv\Scripts\python.exe -m pytest -v
```

Para um arquivo específico:

```powershell
..\venv\Scripts\python.exe -m pytest tests/test_auth_login.py -v
```

Com cobertura HTML em `backend/htmlcov/`:

```powershell
..\venv\Scripts\python.exe -m pytest --cov=app --cov-report=html
start htmlcov\index.html
```

Estado atual: **8/8 testes passando**.

## Como testar o login manualmente

### 1. Via Swagger UI
http://localhost:8000/docs → `POST /api/v1/auth/login` → **Try it out** → body:
```json
{ "login": "SEU_LOGIN", "senha": "<sua senha real do Winthor>" }
```

Sucesso devolve `access_token` + `refresh_token`. Use o `access_token` no botão **Authorize** no topo do Swagger pra testar as rotas protegidas (`/auth/me`, `/coletas-avulsas/`, `/produtos/`, `/qa/logs`, `/produtos/carga/{filial_id}`).

### 2. Via painel
http://localhost:5173/login

- Login válido com perfil → vai pra Home.
- Login válido sem perfil → vai pra `/sem-acesso`.
- Login inválido → mensagem vermelha "Usuário ou senha inválidos".
- Usuário ativo no Winthor mas desativado no Coletor 2001 → 401 (auditável em `error_logs`).

### 3. Via curl

```powershell
curl -X POST http://localhost:8000/api/v1/auth/login `
  -H "Content-Type: application/json" `
  -d "{\"login\":\"SEU_LOGIN\",\"senha\":\"...\"}"
```

## Como testar a sincronização

1. Vai em `/produtos` → **▶️ Rodar agora** (no card amarelo "Agendador automático") — dispara cadastro + estoque das filiais visíveis.
2. Histórico mostra cada execução em tempo real (auto-refresh a cada 2.5s quando há sync em andamento).
3. Sob erro: a mensagem do `SyncLog` traz o motivo (ex.: `234816 saldos descartados — rodar sync de cadastro` ou `12 produtos com valor truncado`).

## Auditoria de dispositivo (`/dispositivos` → 📋 Log)

Cada ação que o app coletor (ou o admin via painel) faz num dispositivo gera linha em `dispositivo_logs`. Acesso pela tela `/dispositivos` → botão **📋 Log** em cada linha — abre drawer lateral com timeline auto-refresh (10s), cor por tipo de evento, detalhes JSON expansíveis.

Útil pra:
- **Investigar "operador reclamou que perdeu coleta"** → procurar evento `arquivar_coleta` ou `logout` com data próxima.
- **Detectar tentativas de invasão** → eventos `login_operador_falhou` em sequência.
- **Auditar exclusões** → mesmo após `DELETE /coletas-arquivadas/{uuid}`, o evento `arquivar_coleta` com `detalhes.apagada=true` + `apagada_por=<admin>` fica preservado.

## Como capturar erros novos

Qualquer exceção não-tratada num endpoint cai no `ErrorLoggingMiddleware` ([middleware.py](../backend/app/core/middleware.py)) e é gravada com:

- `request_id` (também devolvido no header `x-request-id`)
- `method`, `path`, `status_code`
- `exc_type`, `mensagem`, `traceback`
- `usuario_id` (extraído do JWT, se houver)
- `payload` (query + headers, com `Authorization`/`Cookie` mascarados)
- `ip`, `user_agent`

## Estrutura dos testes

```
backend/tests/
├── conftest.py             # seta WINTHOR_DRIVER=stub e JWT_SECRET de teste
├── test_health.py          # /api/v1/health responde 200
├── test_winthor_stub.py    # adapter stub devolve usuário e produtos fake
└── test_auth_login.py      # 5 cenários do fluxo de login + autorização
```

> **Gotcha Windows**: fixtures async que tocam o banco precisam de `await engine.dispose()` antes/depois do `yield` — senão `RuntimeError: Event loop is closed`.

## Endpoints `/qa`

| Método | Rota | Permissão | O que faz |
|--------|------|-----------|-----------|
| GET | `/qa/summary` | `qa.ver` | Totais, últimas 24h, distribuição por status, último erro |
| GET | `/qa/logs?...` | `qa.ver` | Lista filtrável (status_code, method, path_contains, desde, limit, offset) |
| GET | `/qa/logs/{id}` | `qa.ver` | Detalhe com traceback + payload |
| DELETE | `/qa/logs?mais_antigos_que_dias=N` | `admin.usuarios` | Limpeza |

## Limpeza

```powershell
curl -X DELETE "http://localhost:8000/api/v1/qa/logs?mais_antigos_que_dias=30" `
  -H "Authorization: Bearer <seu-token>"
```
