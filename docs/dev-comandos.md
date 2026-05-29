# Comandos de desenvolvimento — Coletor 2001

Guia rápido pra subir, parar e reiniciar os serviços do dia a dia.
Vale pra qualquer um (CLI, PyCharm, ou batch direto no Explorer).

> **Regra geral:** se ficou algo estranho (frontend não loga, "porta ocupada", várias portas 5173/5174/5175...) → rode **`scripts\stop_all.bat`** ou **`scripts\restart_all.bat`** e comece limpo. Esses scripts são cirúrgicos — matam só as portas do projeto, NÃO afetam outros pythons/nodes da máquina.

---

## 1. Scripts em `scripts/` (CLI)

| Script | O que faz | Quando usar |
|---|---|---|
| **`start_all.bat`** | Sobe backend + frontend em 2 janelas separadas | Início do dia. Matam a porta velha antes de subir, então pode rodar de novo sem ter que parar manualmente |
| **`start_backend.bat`** | Só backend (uvicorn em `:8000`, com `--reload`) | Quando só está mexendo em código Python e prefere não ter o vite consumindo CPU |
| **`start_frontend.bat`** | Só frontend (vite em `:5173`, com HMR) | Quando o backend está OK (do `stage`/produção) e você só mexe no React |
| **`stop_all.bat`** | Mata **só as portas do projeto** (8000 e 5173..5180) | Quando algo travou. Não mata outros pythons/nodes da máquina (PyCharm, etc.) |
| **`stop_backend.bat`** | Só porta 8000 | Quando o backend congelou e o vite está OK |
| **`stop_frontend.bat`** | Só portas 5173..5180 | Quando o vite zombiou em várias portas (5174, 5175...) |
| **`restart_all.bat`** | `stop_all` + 2s + `start_all` | Quando você quer "reset hard" sem pensar |
| **`_lib_port.bat`** | Helper interno usado pelos outros | Não chame direto — só os outros scripts usam |

### Exemplo de uso

```cmd
:: Pra começar o dia
scripts\start_all.bat

:: Algo travou — reseta tudo
scripts\restart_all.bat

:: Só quero matar o frontend (vai instalar dep nova e re-subir)
scripts\stop_frontend.bat
cd web && npm install
scripts\start_frontend.bat
```

URLs depois de subir:
- Backend Swagger: <http://localhost:8000/docs>
- Frontend: <http://localhost:5173>

---

## 2. No PyCharm (Run Configurations versionadas)

Existem **6 configurações** em `.idea/runConfigurations/` (commitadas no Git, pra TODOS os devs):

| # | Nome | Tipo | O que faz |
|---|---|---|---|
| 1 | **Backend (uvicorn)** | Shell → `start_backend.bat` | Sobe o backend. ▶ Start e ■ Stop independentes |
| 2 | **Frontend (vite)** | Shell → `start_frontend.bat` | Sobe o vite. ▶ Start e ■ Stop independentes |
| 3 | **Coletor 2001 (full = backend + frontend)** | Compound | Combo das 2 acima. 1 clique sobe tudo, **■ Stop** para os 2 juntos |
| 4 | **Alembic upgrade head** | Python | Aplica migrations pendentes |
| 5 | **Stop All (matar dev servers)** | Shell → `stop_all.bat` | Mata cirurgicamente 8000 e 5173..5180 |
| 6 | **Restart All (stop + start)** | Shell → `restart_all.bat` | Reseta tudo num clique |

### Como usar no PyCharm

1. **Selecionar config**: canto superior direito da janela do PyCharm tem um dropdown com a lista numerada.
2. **▶ Start**: clica no triângulo verde ao lado do dropdown — abre uma aba no painel "Run" (parte de baixo) com o output.
3. **■ Stop**: clica no quadrado vermelho. Mata o processo daquela config — **mas o `stop_all.bat` é mais confiável** porque mata pela porta, garantindo que não fica zombie.
4. **Compound (#3)**: aparece como aba "Run" pai com 2 filhas. Stop nele para os 2 filhos.

### Setup inicial do PyCharm (1x)

Se as Run Configurations não aparecem após `git pull`:
- `File → Reload Project from Disk`
- OU `File → Invalidate Caches... → Just Restart`

Se aparece "SDK invalid": só nas configs 4 (Alembic) pode reclamar do Python SDK. Abra a config → re-selecione o interpreter `Python 3.9 (Coletor_2001)`. As outras configs são Shell, não dependem de SDK.

---

## 3. Casos típicos de "deu ruim"

### "Não consigo logar, frontend abre em http://localhost:5174 (ou 5175, 5176...)"
Você tem zombies do vite ocupando 5173. Tipo o que acabou de acontecer.

**Solução:**
```cmd
scripts\stop_frontend.bat
scripts\start_frontend.bat
```

OU, no PyCharm: Run config **#5 Stop All** → depois #2 Frontend.

### "Backend não responde em http://localhost:8000"
Pode ser que: (a) não está rodando, (b) caiu, (c) zombiou.

**Diagnóstico rápido:**
```cmd
netstat -ano | findstr ":8000.*LISTENING"
```
Se mostra PID: tá rodando. Se vazio: subiu (`scripts\start_backend.bat`).

### "Tudo travou, vou apertar Ctrl+Alt+Del"
Calma. Use o stop cirúrgico:
```cmd
scripts\stop_all.bat
```
Se ainda assim ficar algo (raríssimo), aí sim mata o `node.exe` / `python.exe` específico no Task Manager — mas **NÃO** use o antigo "mata todos pythons" que existia na primeira versão dos scripts. Aquilo derrubaria PyCharm, Anaconda, etc.

### "Alembic upgrade head precisa rodar"
Depois de `git pull` que trouxe migration nova:
```cmd
cd backend
..\venv\Scripts\alembic upgrade head
```
OU no PyCharm: Run config **#4 Alembic upgrade head**.

### "npm install novo pacote"
```cmd
scripts\stop_frontend.bat
cd web
npm install
cd ..
scripts\start_frontend.bat
```

### "Mudei algo no `.env` do backend"
```cmd
scripts\stop_backend.bat
scripts\start_backend.bat
```
(O `--reload` do uvicorn NÃO recarrega `.env`, só código `.py`.)

---

## 4. Diagnóstico — quem está ocupando uma porta?

```cmd
:: Geral — todas as portas com LISTENING
netstat -ano | findstr "LISTENING"

:: Específica
netstat -ano | findstr ":8000.*LISTENING"
netstat -ano | findstr ":5173.*LISTENING"

:: Quem é o PID que apareceu?
tasklist /FI "PID eq 12345"
```

---

## 5. Pré-requisitos (1x)

### Backend
```cmd
cd backend
python -m venv ..\venv     :: se ainda não tem
..\venv\Scripts\activate
pip install -r requirements.txt -r requirements-dev.txt
copy .env.example .env      :: editar com credenciais
..\venv\Scripts\alembic upgrade head
python ..\scripts\seed_dev.py
```

### Frontend
```cmd
cd web
npm install
```

Depois disso, é só `scripts\start_all.bat` (ou Run config #3 no PyCharm).

---

## 6. Resumo visual

```
                      ┌─────────────────────┐
                      │   PyCharm Run       │     CLI equivalente
                      │   Configurations    │     (qualquer shell)
                      └─────────────────────┘
                                │
   ┌────────────────────────────┴─────────────────────────────┐
   │                                                          │
   ▼                                                          ▼
1. Backend                                          start_backend.bat
2. Frontend                                         start_frontend.bat
3. Coletor 2001 (full)  ──────►  Compound  ──────►  start_all.bat
4. Alembic upgrade                                  ..\venv\Scripts\alembic upgrade head
5. Stop All                                         stop_all.bat
6. Restart All                                      restart_all.bat
```

Todos os scripts CLI são **idempotentes** — pode chamar quantas vezes quiser sem efeito colateral. Os `start_*` sempre matam a porta antes de subir, então rodar 2× seguidos não cria zombie.
