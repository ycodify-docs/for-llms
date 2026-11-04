# orgid · `/ua` · associação conta-papel

> Domínio **externo** (`/ua`). Vínculo **conta × papel** (papéis com `owner`/espaço de nomes — ver
> [ua-papel.md](ua-papel.md)). Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. **Papéis
> aceitos:** administrador / engenheiro (no `owner` do papel). É o papel **de plataforma** (`API_MASTER` /
> `API_ENGINEER`) na organização cujo nome é o `owner` do papel, e o token o carrega no claim `orgs` como
> `<owner>/API_MASTER`. **Não** é um papel `/ua`. Erros: [../erros.md](../erros.md).
>
> **A conta `/ua` não pertence a organização nenhuma.** Ela se liga a um dono só pelo papel associado.
>
> **Forma do vínculo no corpo** (recorrente abaixo): `account` `{username}`, `role` `{name, owner}`.

## Contents
- POST /ua/account-role/using/authority — associar papel a conta(s)
- POST /ua/account-role/new-account/using/authority — criar conta já com papel
- PUT /ua/account-role/role-owner/{roleOwner}/using/authority — atualizar associações
- PUT /ua/account-role/status/using/authority — alterar status da associação
- DELETE /ua/account/by/username/{username}/role-name/{roleName}/role-owner/{roleOwner}/using/authority — desassociar

---

## POST /ua/account-role/using/authority
Associa um papel a uma conta (ou várias). **Papéis:** administrador / engenheiro (no dono do papel).

**Corpo** (JSON) — aceita **um objeto** (uma associação) **ou** um **array de objetos** (várias). Cada
objeto tem a forma:

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account` | objeto | **sim** | `{ "username": "…" }` — a conta a associar (`username` obrigatório). |
| `role` | objeto | **sim** | `{ "name": "…", "owner": "…" }` — o papel. **No `/ua`, `owner` é obrigatório** (o espaço de nomes do papel). |
| `status` | string | não | Situação da associação. |

> ⚠️ **Distinção do `/up`:** aqui `role` exige **`owner`** (papel tem dono/namespace). A associação
> da plataforma ([up-associacao.md](up-associacao.md)) usa `role` **sem** `owner` (papéis `API_*`).
> A conta e o papel (`name`+`owner`) **precisam existir** — caso contrário a operação falha.

**Exemplo — objeto único:**
```json
{ "account": { "username": "bob" }, "role": { "name": "cliente", "owner": "acme" } }
```

**Exemplo — array (várias associações):**
```json
[ { "account": { "username": "bob" },   "role": { "name": "cliente", "owner": "acme" } },
  { "account": { "username": "carol" }, "role": { "name": "gestor",  "owner": "acme" } } ]
```

**Resposta:** `200` (sem corpo). Conta ou papel inexistente → **`404`**, dizendo qual dos dois faltou e que
a associação **não foi criada** (`"account not found: the account-role association was not created."` /
`"role not found: …"`); corpo incompleto ou contexto que impede montar a associação → `510` (falha não
tratada — ver [../erros.md](../erros.md)).

## POST /ua/account-role/new-account/using/authority
Cria uma conta `/ua` e já a associa a um papel do dono, numa operação só. O papel pode ser **não público**.
**Papel:** só **administrador** (`API_MASTER`) no `owner` do papel; engenheiro recebe `403`.

> **Ainda não implantada:** está na `develop` do orgid (`orgid@9e16a0b`) e não está na imagem do composer.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account.username` | string | sim | Login. |
| `account.password` | string | sim | Senha, com as mesmas regras do [registro público](publico.md). |
| `account.email` | string | sim | E-mail. |
| `account.*` | — | não | Demais campos da conta externa (ver [ua-conta.md](ua-conta.md)). `account.status` é ignorado. |
| `role.name` | string | sim | Papel a associar. Tem de **existir**; pode ser público ou não. |
| `role.owner` | string | sim | Dono (espaço de nomes) do papel. O portador precisa de `API_MASTER` nele. |

```json
{ "account": { "username": "scriber", "password": "…", "email": "scriber@acme.com" },
  "role": { "name": "ADMIN", "owner": "acme" } }
```

A conta nasce **`ACTIVE`**. Nada é enviado por e-mail.

**Resposta:** `201` — `{ "id": <number> }` (id da conta) · `400` — corpo sem `account`, sem `role` ou sem
`role.owner`, ou campo da conta inválido · `403` — o portador não tem `API_MASTER` no `owner` do papel · `404` —
`"role not found: the account was not created."` · `409` se o `username` já existe. Em toda recusa, **nada**
é gravado: nem conta, nem vínculo.

## PUT /ua/account-role/role-owner/{roleOwner}/using/authority
Atualiza associações em lote para um dono de papel.

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel das associações |

**Corpo** (JSON): **array** de associações `{ account{username}, role{name,owner}, status? }`.

**Resposta:** `200` (sem corpo) · `404` se alguma conta ou papel do lote não existir —
`"… not found: no association was updated."` O lote é tudo-ou-nada: um item ausente aborta a operação
inteira e **nada** é gravado.

## PUT /ua/account-role/status/using/authority
Altera o **status** de uma associação conta-papel. **Corpo é um objeto único** (não array).

**Corpo** (JSON): `account` `{username}` + `role` `{name, owner}` + `status` — `ACTIVE` / `SUSPENDED` / `CANCELED`.

```json
{ "account": { "username": "bob" }, "role": { "name": "cliente", "owner": "acme" }, "status": "SUSPENDED" }
```

**Resposta:** `200` (sem corpo) · `404` — `"account and role association not found: nothing was
updated."`

## DELETE /ua/account/by/username/{username}/role-name/{roleName}/role-owner/{roleOwner}/using/authority
Desassocia um papel de uma conta.

| Path-var | Significado |
|---|---|
| `username` | conta |
| `roleName` | papel a remover |
| `roleOwner` | dono do papel |

**Resposta:** `200` (sem corpo).
