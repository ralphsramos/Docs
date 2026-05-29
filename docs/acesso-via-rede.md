# Acesso via rede (LAN interna) — Coletor 2001

Por padrão o sistema sobe em `localhost` (só a máquina servidor enxerga).
Este guia explica como liberar pra **outras máquinas da rede** acessarem,
e como configurar uma **máscara amigável** (hostname/DNS) pra não ter
que digitar IP.

> ⚠️ **Importante:** isso libera só pra rede **interna** (LAN). Pra
> expor pela internet/VPN/produção é outro nível (precisa Nginx/HTTPS,
> firewall, autenticação adicional). FASE 1 cobre só LAN dev/staging.

---

## 1. Setup base — uma única vez no servidor

### 1.1. Subir backend + frontend escutando em todas as interfaces

Já está configurado:

- **Backend** (`scripts\start_backend.bat`): roda `uvicorn --host 0.0.0.0 --port 8000`
- **Frontend** (`vite.config.ts`): `host: true` (= 0.0.0.0)
- **CORS** (`backend/app/main.py`): regex aceita LAN privada (10.x, 172.16-31.x, 192.168.x) + qualquer hostname sem ponto

### 1.2. Liberar firewall do Windows

Pela primeira vez que rodar, o Windows pergunta. **Marque ambas as opções**
(rede privada + pública) ao subir backend e frontend. Se você clicou em
"Cancelar" por engano, libere manualmente:

```powershell
# Rodar como Administrador
New-NetFirewallRule -DisplayName "Coletor 2001 Backend (8000)" `
  -Direction Inbound -Protocol TCP -LocalPort 8000 -Action Allow
New-NetFirewallRule -DisplayName "Coletor 2001 Frontend (5173)" `
  -Direction Inbound -Protocol TCP -LocalPort 5173 -Action Allow
```

### 1.3. Descobrir o IP do servidor

Na máquina servidor:

```cmd
ipconfig | findstr "IPv4"
```

Exemplo de saída:
```
Endereço IPv4 . . . . . . . . . . . . . . : 192.168.X.X
```

Esse é o **IP do servidor**. Outras máquinas da rede acessam via:

- **Painel web:** `http://192.168.X.X:5173`
- **Swagger API:** `http://192.168.X.X:8000/docs`

### 1.4. (Opcional) Descobrir o hostname do servidor

```cmd
hostname
```

Saída ex: `PC-SERVIDOR`. Por padrão, Windows expõe via NetBIOS — então
qualquer máquina Windows da mesma rede pode acessar via:

- `http://PC-SERVIDOR:5173`
- `http://pc-servidor:5173` (case-insensitive)

Sem precisar configurar DNS. Mas isso só funciona com clientes Windows
e quando a rede permite NetBIOS (a maioria das redes corporativas permite).

---

## 2. Máscara amigável (URL "bonita")

Em vez de digitar `http://192.168.X.X:5173`, queremos algo como
`http://coletor` ou `http://coletor.local`. Tem 3 caminhos:

### 2.A. Hosts file em cada cliente (mais simples, manual)

**Em CADA PC que vai acessar**, edite como **Administrador** o arquivo:

```
C:\Windows\System32\drivers\etc\hosts
```

Adicione no final:

```
192.168.X.X    coletor
192.168.X.X    coletor.local
```

(troque o IP pelo IP real do seu servidor)

Salve. Agora qualquer browser nesse PC acessa via:

- `http://coletor:5173`
- `http://coletor.local:5173`

**Prós:** zero infra. **Contras:** cada PC novo da empresa precisa ser
configurado individualmente.

### 2.B. DNS interno (escalável)

Se a empresa tem servidor DNS interno (AD, pfSense, BIND, OPNsense, etc),
adicione um registro **A** apontando pra IP do servidor:

| Nome | Tipo | Valor |
|---|---|---|
| `coletor.local` | A | `192.168.X.X` |
| `coletor` (atalho) | CNAME | `coletor.local` |

Todos os PCs já enxergam automaticamente — sem mexer em nada no cliente.

### 2.C. mDNS / Bonjour (`.local`)

Windows 10/11 suporta nativamente `<hostname>.local`. Se o servidor se
chama `PC-SERVIDOR`, acessa via `http://pc-servidor.local:5173`. Funciona
sem configurar nada, **mas** depende da rede permitir multicast DNS
(muitas redes corporativas bloqueiam).

---

## 3. Acessando de celular / tablet (rede WiFi)

O app Android (FASE futura) e o **Coletor Virtual** acessado pelo browser
do celular precisam apontar pra IP/hostname do servidor:

1. PC servidor e celular **na mesma rede WiFi**
2. Abre Chrome no celular → `http://192.168.X.X:5173`
   (ou `http://coletor:5173` se configurou hosts/DNS)
3. Faz login normal

⚠️ Se o WiFi tem **"isolamento de clientes"** ativo, dispositivos não
se enxergam entre si. Desligue no painel admin do roteador.

---

## 4. Troubleshooting

### "Esta página não pode ser acessada" / timeout

1. **Firewall**: rode os comandos PowerShell da seção 1.2
2. **Mesma rede**: cliente e servidor estão na mesma sub-rede?
   `ping <ip-servidor>` no cliente
3. **Backend rodando?** Na máquina servidor:
   ```
   netstat -ano | findstr ":8000.*LISTENING"
   ```
   Deve aparecer linha com `0.0.0.0:8000` (não `127.0.0.1:8000`).

### "Carregando perfil... → Não foi possível carregar seu perfil"

1. Backend caiu/travou — reinicie pelo PyCharm (Run Config "Backend")
2. Token JWT expirou (30 min) — clique em "Sair e entrar de novo"
3. Migration recente: rodar `alembic upgrade head` (Run Config #4)

### CORS bloqueando (vejo erro vermelho no DevTools / Network)

O regex no `main.py` aceita LAN privada + qualquer hostname sem ponto.
Se ainda assim bloqueou, adicione o host no `.env`:

```env
CORS_ORIGINS=http://localhost:5173,http://coletor:5173,http://meusite.empresa.com:5173
```

Reinicie o backend.

### Vite escolhe outra porta (5174, 5175...)

Não acontece mais — `strictPort: true` faz falhar em vez de pular.
Se acontecer, mate os zombies: `scripts\stop_frontend.bat`.

---

## 5. Resumo visual

```
                    ┌──────────────────┐
                    │  PC SERVIDOR     │   ipconfig → 192.168.X.X
                    │  Windows + venv  │   hostname → PC-SERVIDOR
                    │                  │
                    │  uvicorn :8000   │ ── 0.0.0.0 ──┐
                    │  vite    :5173   │ ── 0.0.0.0 ──┤
                    │  postgres :5432  │ (só local)   │
                    └──────────────────┘              │
                                                       │ LAN
              ┌─────────────────────┬─────────────────┼─────────────────┐
              │                     │                 │                 │
              ▼                     ▼                 ▼                 ▼
       http://192.168.X.X    http://coletor    http://PC-SERVIDOR    http://coletor.local
            :5173                  :5173             :5173                :5173
       (qualquer cliente)    (clientes com    (Windows clients)   (clientes com DNS
                              hosts file)                          interno OU hosts)
```
