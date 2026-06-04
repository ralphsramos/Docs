# 🔐 Segurança — como o Coletor 2001 se protege

Este documento explica, de forma direta, **como funciona a segurança** do sistema
(autenticação, permissões, transporte, logs) e **quais boas práticas** seguir pra
manter tudo seguro. É um guia pra entender o modelo — não uma lista de problemas.

> Em uma frase: **a barreira de acesso real é o backend.** O menu e as telas do
> front apenas *escondem* o que você não pode usar; quem garante que ninguém vê
> dado que não devia é a API. Toda permissão é checada no servidor.

---

## 1. Camadas de defesa (defense in depth)

A segurança não depende de um único ponto — são camadas que se reforçam:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Transporte   HTTPS (produção) / rede interna confiável     │
│ 2. Autenticação Quem é você?  (login Winthor, JWT, token device)
│ 3. Autorização  O que você pode?  (perfil → permissões)        │
│ 4. Escopo       Quais filiais?  (visibilidade por filial)      │
│ 5. Validação    A entrada é válida?  (schemas, limites, tipos) │
│ 6. Auditoria    O que aconteceu?  (logs de erro + logs do app) │
└─────────────────────────────────────────────────────────────┘
```

Se uma camada falha, a próxima ainda segura. Ex.: mesmo que alguém descubra a URL
de uma página administrativa (camada 1/2), a API nega os dados sem a permissão
(camada 3) e fora da filial dele (camada 4).

---

## 2. Autenticação — "quem é você?"

| Ator | Como autentica | Onde mora a senha |
|---|---|---|
| **Usuário (web/app)** | Login validado no **Winthor (Oracle)** via procedure do próprio ERP | No ERP — o sistema **nunca** guarda a senha do Winthor |
| **Conta de sistema** | Login local com **bcrypt** (fora do Winthor), pra automação/superusuário | Hash bcrypt no banco; a senha em claro só no `.env` (fora do git) |
| **Dispositivo (coletor)** | Cabeçalho `X-Device-Token` emitido no pareamento | Token no banco; o aparelho guarda só o token |

Depois do login, o usuário recebe um **JWT** (JSON Web Token):

- **Access token** — curto, usado em cada requisição (`Authorization: Bearer …`).
- **Refresh token** — mais longo, para renovar a sessão.
- O token de **acesso** e o de **refresh** são distintos: um não vale pelo outro
  (o backend exige `type == "access"` nas rotas protegidas).
- Desativar um usuário tem **efeito imediato**: o backend recarrega o usuário do
  banco a cada requisição — conta inativa = barrada na hora.

---

## 3. Autorização — "o que você pode?"

O acesso é por **perfil → permissões**. Um usuário recebe um ou mais **perfis**
(em uma ou todas as filiais), e cada perfil carrega um conjunto de **permissões**
(chaves como `produtos.ver`, `coletas_avulsas.ver`, `qa.ver`, `admin.usuarios`).

- No backend, cada endpoint declara a permissão que exige. Sem ela → **403**.
- Sem nenhum perfil atribuído → **403** ("Sem perfil atribuído").
- O menu do front só mostra o item se o usuário tem a permissão — mas isso é
  **conveniência**, não segurança: a API é quem bloqueia de verdade.

Detalhe das permissões e de como atribuir está em
[permissoes-e-filiais.md](permissoes-e-filiais.md).

### Escopo por filial

Além da permissão, há a **visibilidade por filial**: um usuário só enxerga as
filiais às quais foi vinculado (ou todas, se tiver acesso total). Os dados de
estoque, coletas, inventário e impressão são filtrados por filial no servidor.

---

## 4. Boas práticas que o sistema já adota

- **Anti-SQL injection:** todas as consultas usam *bind parameters* — nunca se
  monta SQL concatenando valor de usuário.
- **Senha protegida:** a senha do ERP nunca é registrada em log nem devolvida em
  resposta; a conta local usa **bcrypt** com salt.
- **Segredos fora do código:** chaves e credenciais ficam só no `.env` (que é
  *gitignored*); o repositório versiona apenas um `.env.example` com marcadores.
- **Logs sem dado sensível:** o middleware de erro remove cabeçalhos sensíveis
  (`Authorization`, `Cookie`) antes de gravar.
- **Idempotência:** cada leitura do coletor tem um id local único; reenviar o
  mesmo lote **não duplica** dados no servidor.
- **Upload seguro:** uploads (planilhas, binários) têm limite de tamanho,
  validação de tipo e proteção contra *path traversal* no nome do arquivo.

---

## 5. Configuração segura (checklist do ambiente)

O que o responsável pelo ambiente deve garantir, especialmente em **produção**:

- [ ] **`JWT_SECRET` forte e único** (≥ 32 caracteres aleatórios). Nunca use o
      valor de exemplo. Trocar o secret invalida todas as sessões.
- [ ] **HTTPS em produção** — especialmente para o tráfego do coletor (o token do
      dispositivo viaja no cabeçalho). Em rede interna, mantenha-a confiável.
- [ ] **CORS restrito** às origens realmente usadas (painel web / app), não a
      qualquer host.
- [ ] **`APP_DEBUG=false`** em produção e documentação interativa da API
      (`/docs`, `/redoc`) desligada ou restrita.
- [ ] **Rede do coletor confiável** — o token do dispositivo é uma credencial;
      configure o servidor do app apenas para endereços conhecidos.
- [ ] **Backups e rotação** — rotacione tokens de dispositivos perdidos
      (a tela de Dispositivos permite); desative usuários que saíram.
- [ ] **Atualizações (OTA)** servidas de origem confiável e conferidas por
      checksum antes de aplicar.

---

## 6. QA de segurança — testes automatizados + observabilidade

### Logs de erro (`/qa`)

Todo erro 5xx (e 401/403 selecionados) é capturado por um middleware e gravado na
tabela `error_logs`, com `request_id` pra rastreio. A página **QA / Logs** mostra
um resumo (últimas 24h, por status, caminhos mais afetados) e o detalhe de cada
erro. O acesso à página exige a permissão `qa.ver` (ou ser admin).

### Logs do app (coletor)

O app mantém um buffer de eventos (pareamento, login do operador, sincronização,
erros de JavaScript capturados globalmente) que **sobe junto no SINCRONIZAR** —
permite diagnosticar um problema no aparelho remotamente, sem o operador copiar
nada. A senha do operador **nunca** entra nesse log.

### Suíte de testes de segurança

Há uma bateria automatizada que **trava os controles de acesso** (e detecta
regressão se alguém afrouxar um gate sem querer):

```bash
cd backend
# Windows (venv do projeto):
..\venv\Scripts\python.exe -m pytest tests/test_seg_*.py -v
# Tudo:
python -m pytest -q
```

O que ela cobre:

| Arquivo | Garante que… |
|---|---|
| `test_seg_acesso.py` | sem token → 401; com perfil mas sem a permissão → 403; com a permissão → 200 (para produtos, coletas avulsas e QA) |
| `test_seg_auth.py` | token inválido e *refresh* são rejeitados em rota de acesso; CORS reflete só origem legítima |
| `test_seg_device.py` | os endpoints `/device/*` exigem um `X-Device-Token` válido |

Os testes rodam com o adaptador Winthor em modo *stub* (não tocam o ERP real) e
criam/limpam seus próprios dados de teste (prefixo `PYTEST_SEC_`).

---

## 7. Princípios para evoluir com segurança

1. **Permissão é no backend.** Esconder no front é UX; todo endpoint novo declara
   a permissão que exige.
2. **Filtre por filial** em toda rota que recebe um id — não basta filtrar a
   listagem; o detalhe e o *delete* também precisam.
3. **Valide a entrada** com schema (tipos, limites de tamanho) — nunca confie no
   cliente.
4. **Não logue segredo.** Token, senha e payload sensível ficam fora do log.
5. **Idempotência** em qualquer transmissão que possa ser reenviada.
6. **Cada correção de segurança ganha (ou atualiza) um teste** em `test_seg_*` —
   o que está protegido fica protegido.

> Dúvidas sobre o modelo de permissões e filiais? Veja
> [permissoes-e-filiais.md](permissoes-e-filiais.md). Sobre a arquitetura geral,
> [arquitetura.md](arquitetura.md).
