# orgid · público (`/open/*`)

> Endpoints **públicos por design** (sem autenticação): verificação de existência, **registro** e fluxo
> de **hash** (ativação / recuperação de senha), nos dois domínios (`/up` plataforma, `/ua` externo).
> Distintos dos endpoints internos ofuscados (que **não** são documentados). Guia: [../README.md](../README.md).
>
> **Comum:** sem `Authorization`; POST/PUT enviam `Content-Type: application/json`.
> `{action}`: **`R`** = ativação (registro) · **`PR`** = recuperação de senha. Erros: [../erros.md](../erros.md).
>
> ⚠️ **O prefixo `/open/` não é o inventário completo dos públicos.** Existe **um** público fora dele —
> `GET /ua/open/role/owner/{roleOwner}`, com os segmentos invertidos (ver [ua-papel.md](ua-papel.md)).
> Quem deriva regra de autorização do prefixo `/open/**` **deixa esse de fora** e ele passa a exigir token
> apesar de público. É o estado do `OrgIdSecurityConfig` do orgid standalone até hoje; o `composer`, que é
> o que está implantado, libera os dois prefixos desde 2026-09-05.

## Contents
- GET /open/up/account/by/username/{username}/exists
- GET /open/ua/account/by/username/{username}/exists
- POST /open/up/account — registro: conta + organização (plataforma)
- POST /open/ua/account-role — registro: conta + papel (externo)
- GET /open/{up|ua}/hash/for/{action}/by/{username} — solicitar hash
- PUT /open/{up|ua}/account/{username}/{action}/hash/{hash} — executar ação
- POST /open/ua/account/e-mail/send — enviar e-mail (externo)
- GET /ua/open/role/owner/{roleOwner} — listar papéis públicos ⚠️ **fora do prefixo** → [ua-papel.md](ua-papel.md)

---

## ⚠️ Registro não envia e-mail; hash e `/e-mail/send` enviam — e o envio é real

`POST /open/up/account` e `POST /open/ua/account-role` **criam a conta e não disparam nada**. Quem manda
e-mail é `GET /open/{up|ua}/hash/for/{action}/by/{username}` (o hash vai por e-mail) e
`POST /open/ua/account/e-mail/send`.

Esses dois falam com um **provedor real** (mailersend), em plano com **cota**. Não os acione para exercitar
fluxo: cada chamada consome cota e reputação de domínio. Para testar registro sem enviar nada, pare no POST
da conta — ela já existe e, como diz [ua-conta.md](ua-conta.md), **já dá para logar sem ativar**.

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

**Resposta:** `200` — `{}` (efeito: envio do hash) · `404` — `"account not found: no hash was generated
and no e-mail was sent."`

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

**Resposta:** `200` (sem corpo) — em `R`, ativa a conta; em `PR`, troca a senha · `404` —
`"account not found: the hash action was not executed."`

> ⚠️ Até 2026-09-05 a conta inexistente respondia **`204`** nos dois endpoints acima. `204` é sucesso sem
> corpo: em `PR`, o usuário recebia confirmação de uma troca de senha que nunca aconteceu. Ver
> [../erros.md](../erros.md).

## POST /open/ua/account/e-mail/send — enviar e-mail (externo)
Envia uma notificação por e-mail.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `to` | string | sim | Destinatário. |
| `subject` | string | sim | Assunto. |
| `body` | string | sim | Corpo (texto). |

**Resposta:** `200` — `{}`.
