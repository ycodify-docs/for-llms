# orgid · público (`/open/*`)

> Endpoints **públicos por design** (sem autenticação): verificação de existência, **registro** e fluxo
> de **hash** (ativação / recuperação de senha), nos dois domínios (`/up` plataforma, `/ua` externo).
> Distintos dos endpoints internos ofuscados (que **não** são documentados). Guia: [../README.md](../README.md).
>
> **Comum:** sem `Authorization`; POST/PUT enviam `Content-Type: application/json`.
> `{action}`: **`R`** = ativação (registro) · **`PR`** = recuperação de senha. Erros: [../erros.md](../erros.md).

## Contents
- GET /open/up/account/by/username/{username}/exists
- GET /open/ua/account/by/username/{username}/exists
- POST /open/up/account — registro: conta + organização (plataforma)
- POST /open/ua/account-role — registro: conta + papel (externo)
- GET /open/{up|ua}/hash/for/{action}/by/{username} — solicitar hash
- PUT /open/{up|ua}/account/{username}/{action}/hash/{hash} — executar ação
- POST /open/ua/account/e-mail/send — enviar e-mail (externo)

---

## GET /open/up/account/by/username/{username}/exists
## GET /open/ua/account/by/username/{username}/exists
Verifica existência de conta (plataforma / externo).

| Path-var | Significado |
|---|---|
| `username` | conta a verificar |

**Resposta:** `200` (existe) · `204` (não existe). Sem corpo.

## POST /open/up/account — registro (plataforma): conta + organização
Cria a **conta** (estado inicial **pendente**) **e** a **organização**, vinculando a conta como
**administradora**. É o ponto de entrada de um novo cliente na plataforma (antes do forger).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account.username` | string | sim | Login da conta. |
| `account.password` | string | sim | Senha. |
| `account.email` | string | sim | E-mail. |
| `account.name` | string | não | Nome. |
| `account.status` | string | não | Padrão `PENDING`. |
| `account.end*` | string | não | Endereço (rua/número/…/cep). |
| `org.name` | string | sim | Nome da organização (vira o `{org}`). |
| `org.client` | string | sim | **Nome da empresa cliente** que contratou a plataforma (toda org sinaliza o seu). Máx. 12 caracteres. |
| `org.alias` | string | sim | Apelido da org. |

**Resposta:** `201` — conta criada (com `id`; senha omitida). Depois: **ativar** (fluxo de hash) e seguir para o forger.

## POST /open/ua/account-role — registro (externo): conta + papel
Cria uma conta **externa** e a associa a um **papel**.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account.username` | string | sim | Login. |
| `account.password` | string | sim | Senha. |
| `account.email` | string | sim | E-mail. |
| `account.*` | — | não | Demais campos da conta externa (ver [ua-conta.md](ua-conta.md)). |
| `role.name` | string | sim | Papel a associar. |
| `role.owner` | string | sim | Dono (espaço de nomes) do papel. |
| `role.label` / `role.ispublic` / `role.status` | — | não | Metadados do papel. |

**Resposta:** `200` (sem corpo).

## GET /open/up/hash/for/{action}/by/{username}
## GET /open/ua/hash/for/{action}/by/{username}
Solicita um **hash** de vida curta (enviado ao usuário por e-mail) para `{action}` (`R` ativação / `PR` recuperação).

| Path-var | Significado |
|---|---|
| `action` | `R` (ativação) ou `PR` (recuperação de senha) |
| `username` | conta alvo |

**Resposta:** `200` — `{}` (efeito: envio do hash).

## PUT /open/up/account/{username}/{action}/hash/{hash}
## PUT /open/ua/account/{username}/{action}/hash/{hash}
Executa a ação do hash.

| Path-var | Significado |
|---|---|
| `username` | conta |
| `action` | `R` (ativa a conta → `ACTIVE`) ou `PR` (redefine a senha) |
| `hash` | hash recebido por e-mail (uso único, vida curta) |

**Corpo** (JSON) — apenas para `PR`:

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `password` | string | sim (em `PR`) | Nova senha. |

**Resposta:** `200` (sem corpo). Em `R`, ativa a conta; em `PR`, troca a senha.

## POST /open/ua/account/e-mail/send — enviar e-mail (externo)
Envia uma notificação por e-mail.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `to` | string | sim | Destinatário. |
| `subject` | string | sim | Assunto. |
| `body` | string | sim | Corpo (texto). |

**Resposta:** `200` — `{}`.
