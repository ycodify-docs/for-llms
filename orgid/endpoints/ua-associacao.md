# orgid · `/ua` · associação conta-papel

> Domínio **externo** (`/ua`). Vínculo **conta × papel** (papéis com `owner`/espaço de nomes — ver
> [ua-papel.md](ua-papel.md)). Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. **Papéis
> aceitos:** administrador / engenheiro (no `owner` do papel). Erros: [../erros.md](../erros.md).
>
> **Forma do vínculo no corpo** (recorrente abaixo): `account` `{username}`, `role` `{name, owner}`.

## Contents
- POST /ua/account-role/using/authority — associar papel a conta(s)
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

**Resposta:** `200` (sem corpo). Conta/papel inexistentes → `204`/`400`; corpo incompleto ou contexto
que impede montar a associação → `510` (falha não tratada — ver [../erros.md](../erros.md)).

## PUT /ua/account-role/role-owner/{roleOwner}/using/authority
Atualiza associações em lote para um dono de papel.

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel das associações |

**Corpo** (JSON): **array** de associações `{ account{username}, role{name,owner}, status? }`.

**Resposta:** `200` (sem corpo).

## PUT /ua/account-role/status/using/authority
Altera o **status** de uma associação conta-papel. **Corpo é um objeto único** (não array).

**Corpo** (JSON): `account` `{username}` + `role` `{name, owner}` + `status` — `ACTIVE` / `SUSPENDED` / `CANCELED`.

```json
{ "account": { "username": "bob" }, "role": { "name": "cliente", "owner": "acme" }, "status": "SUSPENDED" }
```

**Resposta:** `200` (sem corpo).

## DELETE /ua/account/by/username/{username}/role-name/{roleName}/role-owner/{roleOwner}/using/authority
Desassocia um papel de uma conta.

| Path-var | Significado |
|---|---|
| `username` | conta |
| `roleName` | papel a remover |
| `roleOwner` | dono do papel |

**Resposta:** `200` (sem corpo).
