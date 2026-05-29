# Dev vs Produção — Workflow Paralelo

Estratégia: **dois diretórios paralelos na mesma máquina**, cada um com seu
banco PostgreSQL, venv Python e build. Dev fica sempre na branch `develop`
(ou qualquer feature branch), prod sempre na `main`. Deploy = git pull +
alembic + build + restart.

## Topologia

```
Dev (uvicorn --reload + vite HMR)
├─ C:\Projetos\Coletor_2001
├─ venv: C:\Projetos\Coletor_2001\venv
├─ Banco: Coletor_2001
├─ Backend: :8150
└─ Frontend: :8152

Produção (uvicorn workers + estáticos servidos pelo backend)
├─ C:\Coletor_2001_PROD
├─ venv: C:\Coletor_2001_PROD\venv
├─ Banco: Coletor_2001_prod
└─ Tudo em :8151 (backend + SPA single-port)

Range alocado pela app: 8150-8160 (ver docs/portas.md).
```

Ambos compartilham: **Oracle Winthor** (mesma conexão) — porque é fonte de
dados externa read-only. A separação é só dos bancos próprios PostgreSQL.

## Setup inicial (uma vez)

### 1. Cria o banco de produção no PostgreSQL

Pelo **pgAdmin** (ou psql como superuser):

```sql
CREATE DATABASE "Coletor_2001_prod" OWNER python;
```

Ou via psql com superuser:
```bash
"C:\Program Files\PostgreSQL\17\bin\psql.exe" -h localhost -U postgres -c "CREATE DATABASE \"Coletor_2001_prod\" OWNER python;"
```

### 2. Roda o script de setup prod

```bat
cd C:\Projetos\Coletor_2001
scripts\prod\setup_prod_inicial.bat
```

Esse script:
1. Clona o repo dev em `C:\Coletor_2001_PROD`
2. Cria venv próprio
3. Instala dependencies
4. Copia `.env` ajustando `DATABASE_URL` pro `_prod` + setando `APP_ENV=prod`,
   `APP_DEBUG=false`, `SERVE_STATIC=true`, `STATIC_DIR=...\web\dist`
5. Roda `alembic upgrade head` no banco _prod (cria todas as tabelas)
6. Faz `npm install` + `npm run build` em `web/dist`

### 3. Sobe os 2 ambientes

```bat
REM Em uma janela:
scripts\dev\start_dev.bat
REM   → http://localhost:8000 (backend) + http://localhost:5173 (vite)

REM Em outra janela:
C:\Coletor_2001_PROD\scripts\prod\start_prod.bat
REM   → http://localhost:8080 (backend + SPA tudo junto)
```

## Workflow diário

### Desenvolver em dev

Trabalha normal no `C:\Projetos\Coletor_2001`. Branch
`develop` ou feature branch. HMR + uvicorn reload. Testes. Quando fechar
um chunk:

```bash
git checkout develop
git add . && git commit -m "feat: nova aba X"
git push origin develop

# Quando tiver pronto pra prod, merge na main:
git checkout main
git merge develop --no-ff
git push origin main
```

### Deploy em prod

Na máquina prod (mesma máquina nesse setup):

```bat
REM 1. Para o prod (Ctrl+C na janela do start_prod.bat)
REM 2. Roda deploy:
C:\Coletor_2001_PROD\scripts\prod\deploy.bat
REM    → git pull origin main + alembic + npm build
REM 3. Sobe de novo:
C:\Coletor_2001_PROD\scripts\prod\start_prod.bat
```

### Rollback rápido

Se algo quebrou em prod:

```bat
cd C:\Coletor_2001_PROD
git log --oneline -10           REM ve os últimos commits
git checkout <sha-anterior>     REM volta o codigo
REM Se migration foi destrutiva, ainda precisa rodar:
backend\venv\Scripts\python.exe -m alembic downgrade -1
REM Rebuilda:
cd web && npm run build
REM Sobe:
scripts\prod\start_prod.bat
```

> ⚠ Pra evitar dores de cabeça com rollback de migration, **prefira
> migrations aditivas** (sempre adicionar coluna nullable, depois popular,
> depois tornar NOT NULL noutra migration). Drop só quando 100% certo.

## Como o backend sabe se está em dev ou prod

Via env vars no `.env`:

| Var | Dev | Prod |
|---|---|---|
| `APP_ENV` | `dev` | `prod` |
| `APP_DEBUG` | `true` | `false` |
| `APP_PORT` | `8150` | `8151` |
| `APP_HOST` | `0.0.0.0` | `0.0.0.0` |
| `DATABASE_URL` | `.../Coletor_2001` | `.../Coletor_2001_prod` |
| `SERVE_STATIC` | `false` | `true` |
| `STATIC_DIR` | (vazio) | `C:\Coletor_2001_PROD\web\dist` |

E no frontend, `web/.env`:

| Var | Dev | Prod |
|---|---|---|
| `VITE_DEV_PORT` | `8152` | (não usado em prod — build estático) |
| `VITE_API_BASE_URL` | `http://localhost:8150/api/v1` | `/api/v1` (single-port) ou `http://192.168.X.X:8151/api/v1` |

Os scripts `start_dev.bat` e `start_prod.bat` **leem `APP_PORT` e `APP_HOST` do `.env`** automaticamente — então pra trocar de porta basta editar o `.env`, sem mexer em batch.

Quando `SERVE_STATIC=true`, `app/main.py` monta:
- `GET /assets/*` → serve `web/dist/assets/*` (cache longo)
- `GET /{qualquer-outra-coisa}` → serve `web/dist/index.html` (SPA fallback)
- `POST /api/v1/*` → API normal (resolvida antes da SPA)

## Bancos isolados — como NÃO confundir

A regra é só uma: **NUNCA aponte o `.env` de dev pro `_prod`** e vice-versa.
Pra reduzir risco humano:

1. O `setup_prod_inicial.bat` já ajusta automaticamente
2. Backend escreve no log de startup qual env+banco subiu:
   ```
   {"event": "startup", "env": "prod", "version": "0.4.0"}
   ```
3. Frontend tem badge "DEV" no canto quando `APP_ENV != prod` (a fazer)

## CI/CD futuro

O fluxo manual desta doc tá pensado pra **mão na massa rápida**. Caminho
natural de evolução (quando justificar):

1. **GitHub Actions** dispara em push pra `main`:
   - Roda testes
   - SSH no servidor prod
   - Executa `deploy.bat`
   - Health check
2. **Windows Service** (NSSM) pra subir `start_prod.bat` automaticamente
   ao boot (ver `scripts/prod/install_as_service.bat` — TODO)
3. **Blue/green** com 2 instâncias prod (porta 8080 e 8081) + nginx/Caddy
   alternando — zero downtime no deploy
4. **Docker Compose** quando quiser portabilidade pra outro host

## FAQ

**P: E se eu quiser testar com dados reais no dev?**
R: Tire um backup do `_prod` e restore no `Coletor_2001` (dev):
```bash
pg_dump -h localhost -U python Coletor_2001_prod > backup.sql
psql -h localhost -U python Coletor_2001 < backup.sql
```

**P: Como sincronizo configs entre dev e prod?**
R: O `.env` do prod é gerado a partir do dev (substituindo só DATABASE_URL).
Configs no banco (tabela `app_config`) ficam isoladas — cada ambiente tem
as suas. Migrate manual via `INSERT ... SELECT` se precisar.

**P: Tags de release?**
R: Recomendado pra produção. Antes de deploy:
```bash
git tag v0.4.1
git push origin v0.4.1
```
O `deploy.bat` pega da `main` por enquanto — você pode editar pra
`git checkout v0.4.1` se quiser pin de versão.
