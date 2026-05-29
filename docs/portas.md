# Alocação de portas TCP — Coletor 2001

Range reservado pela aplicação: **8150-8160** (faixa "user ports" alta, livre
de IANA-assigned services típicos do Windows).

## Alocação atual

| Porta | Ambiente | Serviço | Configuração |
|-------|----------|---------|--------------|
| **8150** | DEV | Backend FastAPI (uvicorn --reload) | `backend/.env` → `APP_PORT=8150` |
| **8151** | PROD | Backend FastAPI (uvicorn workers) | `C:\Coletor_2001_PROD\backend\.env` → `APP_PORT=8151` |
| **8152** | DEV | Frontend Vite HMR | `web/.env` → `VITE_DEV_PORT=8152` |
| **8153** | PROD | (reservado — single-port deploy serve UI no 8151) | — |
| **8154** | DEV | Mock printer alternativo (default 9100 mantido) | `python scripts/mock_printer.py --porta 8154` |
| **8155-8160** | — | Reservadas: workers extras, blue/green, WebSocket dedicado, smoke tests | — |

## Outras portas usadas (fora do range)

| Porta | Serviço | Notas |
|-------|---------|-------|
| 5432 | PostgreSQL | Padrão Postgres, instalação local |
| 1521 | Oracle Winthor | Padrão Oracle, conexão remota ao ERP |
| 9100 | Impressoras térmicas | **Padrão indústria** (Zebra/Argox/Elgin); mock pode subir aqui pra simular real |

## Por que esse range?

- **Livre de conflitos comuns**: 80/443 (web), 5432 (Postgres), 8080 (proxies),
  3000 (Node default), 5000 (Flask), 8000 (Django/FastAPI default)
- **Continuidade visual**: tudo na mesma faixa facilita firewall rules,
  monitoring, debugging com `netstat`
- **Espaço pra crescer**: 11 portas dão sobra pra blue/green deployments,
  workers separados (Celery), métricas (Prometheus exporter), etc.

## Como mudar uma porta

**Backend (dev ou prod):**
```bash
# Edite o .env do ambiente:
APP_PORT=8155

# Os scripts batch start_dev.bat / start_prod.bat leem o .env
# automaticamente — basta reiniciar.
```

**Frontend (vite dev):**
```bash
# web/.env:
VITE_DEV_PORT=8156

# E ajuste a URL da API se o backend trocou:
VITE_API_BASE_URL=http://localhost:8155/api/v1
```

**Verificar disponibilidade antes de mudar:**
```bash
# Windows PowerShell:
netstat -ano | findstr ":815"

# Git Bash:
netstat -ano | grep -E ":815[0-9]"
```

## Acesso pela LAN

Os usuários acessam o ambiente PROD pelo IP do servidor:
```
http://<ip-servidor>:8151/
```

Acesso via DNS interno ou máscara (ver `docs/acesso-via-rede.md`):
```
http://coletor:8151/
http://coletor.local:8151/
```

## Firewall

Pra liberar acesso da LAN:

```powershell
# Windows Firewall (rodar como Admin):
New-NetFirewallRule -DisplayName "Coletor 2001 PROD" -Direction Inbound -LocalPort 8151 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Coletor 2001 DEV"  -Direction Inbound -LocalPort 8150,8152 -Protocol TCP -Action Allow
```
