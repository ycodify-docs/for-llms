# orgid · `/up` · associação conta-papel-org

> Domínio **plataforma** (`/up`). Vínculo **tripla** conta × papel × organização — base da autorização
> multi-tenant: define **qual conta tem qual papel em qual org**. Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. Autorização
> **na organização** do vínculo. Erros: [../erros.md](../erros.md).
>
> **Forma do vínculo no corpo** (recorrente abaixo): `account` `{username}`, `role` `{name}` (papel
> `API_MASTER`/`API_ENGINEER`/`API_ANALYST`/`API_FINANCIAL`), `org` `{name}`.

## Contents
- POST /up/account-role-org — criar vínculo
- PUT /up/account-role-org/account-status — alterar status do vínculo
- PUT /up/account-role-org/status/by-master — alterar status (override)
- PUT /up/account-role-org/replace-account — trocar a conta
- PUT /up/account-role-org/replace-master-account — trocar a conta administradora
- PUT /up/account-role-org/replace-role — trocar o papel
- DELETE /up/account-role-org/by/org/{orgName} — desassociar da org

---

## POST /up/account-role-org
Cria o vínculo conta-papel-org. **Papéis:** administrador / engenheiro / analista / financeiro.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account` | objeto | sim | `{ "username": "…" }` — conta a vincular. |
| `role` | objeto | sim | `{ "name": "API_…" }` — papel atribuído. |
| `org` | objeto | sim | `{ "name": "…" }` — organização do vínculo. |

**Resposta:** `201` (sem corpo). `403` sem papel na org.

## PUT /up/account-role-org/account-status
Altera o **status** de um vínculo (o `account.username` é forçado ao do token).

**Corpo** (JSON): `status` (string) + `account` `{username}` + `role` `{name}` + `org` `{name}`.

**Resposta:** `200` (sem corpo).

## PUT /up/account-role-org/status/by-master
Altera o status do vínculo por um administrador (mesma forma de corpo do anterior; sem forçar o username).

**Corpo** (JSON): `status` + `account` `{username}` + `role` `{name}` + `org` `{name}`.

**Resposta:** `200`.

## PUT /up/account-role-org/replace-account
Substitui **a conta** de um vínculo existente. **Papel:** administrador na org.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `org` | objeto | sim | `{ "name": "…" }` — org do vínculo. |
| `accountRoleOrg` | objeto | sim | Vínculo atual a alterar. |
| `account` | objeto | sim | Nova conta (`{username}`). |

**Resposta:** `200`.

## PUT /up/account-role-org/replace-master-account
Substitui a **conta administradora** do vínculo. **Papel:** administrador na org. Corpo igual ao
`replace-account`. **Resposta:** `200`.

## PUT /up/account-role-org/replace-role
Substitui o **papel** do vínculo. **Papel:** administrador na org.

**Corpo** (JSON): `org` `{name}` + `accountRoleOrg` (vínculo atual) + `role` `{name}` (novo papel).

**Resposta:** `200`.

## DELETE /up/account-role-org/by/org/{orgName}
Desassocia a conta da org. **Papel:** administrador na org.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização da qual desassociar |

**Resposta:** `200` (sem corpo).
