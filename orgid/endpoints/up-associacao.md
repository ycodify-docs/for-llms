# orgid · `/up` · associação conta-papel-org

> Domínio **plataforma** (`/up`). Vínculo **tripla** conta × papel × organização — base da autorização
> multi-tenant: define **qual conta tem qual papel em qual org**. Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. Autorização
> **na organização** do vínculo. Erros: [../erros.md](../erros.md).
>
> **Forma do vínculo no corpo** (recorrente abaixo): `account` `{username}`, `role` `{name}`, `org` `{name}`.
> Papéis que se atribuem por aqui: `API_ENGINEER`, `API_ANALYST`, `API_FINANCIAL`, `API_GUEST`,
> `SKOS_MASTER` e `SKOS_ANALYST`. O `API_MASTER` **não** se atribui por aqui: nasce com a organização, no
> registro ([publico.md](publico.md)) ou no `POST /up/org`.

## Contents
- Quem gere cada papel
- POST /up/account-role-org — criar vínculo
- PUT /up/account-role-org/account-status — alterar status do vínculo
- PUT /up/account-role-org/status/by-master — alterar status (override)
- PUT /up/account-role-org/replace-account — trocar a conta
- PUT /up/account-role-org/replace-master-account — trocar a conta administradora
- PUT /up/account-role-org/replace-role — trocar o papel
- DELETE /up/account-role-org/by/org/{orgName} — desassociar da org

---

## Quem gere cada papel

Vale para **criar** o vínculo (`POST /up/account-role-org`) e para **mudar o status** dele por terceiro
(`PUT /up/account-role-org/status/by-master`).

| Papel do vínculo | Quem gere | Como a organização é identificada |
|---|---|---|
| `API_ENGINEER`, `API_ANALYST`, `API_FINANCIAL`, `API_GUEST` | o **dono** da organização | `org.name`, entre as organizações de quem chama |
| `SKOS_MASTER` | o **dono** da organização | `org.name`, entre as organizações de quem chama |
| `SKOS_ANALYST` | quem tem `SKOS_MASTER` **ativo** naquela organização | `org.name` + `org.owner` |

- **`SKOS_MASTER` e `SKOS_ANALYST`** são papéis comuns a mais de um módulo que roda sobre a plataforma (a
  base de conhecimento, `yc.kb`, e o KORC). O orgid só os atribui; o que cada um permite é definido pelo
  módulo que os lê do token.
- O dono da organização **não** atribui `SKOS_ANALYST` só por ser dono: precisa ter também `SKOS_MASTER`
  ativo nela, e pode atribuí-lo a si mesmo.
- "Ativo" é o vínculo `SKOS_MASTER` de quem chama com `accountStatus` e `orgStatus` em `ACTIVE`, lido do
  cadastro **no momento da chamada**, e não do token.
- Quem tem `SKOS_MASTER` não gere outro `SKOS_MASTER`, nem os papéis `API_*`.
- No token do login, o vínculo aparece no claim `orgs` como `<org>/SKOS_MASTER` ou `<org>/SKOS_ANALYST`, e
  em `authorities` como `ROLE_SKOS_MASTER` ou `ROLE_SKOS_ANALYST`, do mesmo modo que os papéis `API_*`.
  Vínculo criado depois do login só entra no próximo token.
- O vínculo só entra no token com **quatro status em `ACTIVE`**: o do papel, o da organização e os dois do
  vínculo (`accountStatus` e `orgStatus`). Com qualquer um diferente ele fica de fora, e o login não acusa
  erro.

> Em produção desde 2026-10-07T17:06Z. Até então, `POST /up/account-role-org` e as duas rotas de status
> respondiam `400` a `SKOS_MASTER` e `SKOS_ANALYST`.

## POST /up/account-role-org
Cria o vínculo conta-papel-org. **Quem chama:** token com papel `API_MASTER`, `API_ENGINEER`,
`API_ANALYST`, `API_FINANCIAL` ou `SKOS_MASTER`; e ainda tem de ser quem **gere** o papel pedido
([Quem gere cada papel](#quem-gere-cada-papel)).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account` | objeto | sim | `{ "username": "…" }` — conta a vincular. Tem de existir. |
| `role` | objeto | sim | `{ "name": "…" }` — `API_ENGINEER`, `API_ANALYST`, `API_FINANCIAL`, `API_GUEST`, `SKOS_MASTER` ou `SKOS_ANALYST`. |
| `org.name` | string | sim | Organização do vínculo. |
| `org.owner` | string | não | Dono da organização. Só é lido quando o papel é `SKOS_ANALYST`; sem ele, vale quem chama. |
| `accountStatus`, `orgStatus` | string | não | Status inicial do vínculo. Padrão `ACTIVE`. |

**Resposta:** `201` (sem corpo) · `400` — `"Papel desconhecido ou não autorizado."` (nome fora da lista,
inclusive `API_MASTER`) · `400` se a conta já tem esse papel nessa organização · `403` — quem chama não tem
`SKOS_MASTER` ativo na organização e pediu `SKOS_ANALYST` · `404` — `"conta de usuário não existe: o vínculo
não foi criado."` ou `"organização não existe para este dono: o vínculo não foi criado."`

## PUT /up/account-role-org/account-status
A **própria conta** suspende ou cancela o seu vínculo (o `account.username` é forçado ao do token).
**Quem chama:** token com papel `API_MASTER`, `API_ENGINEER`, `API_ANALYST`, `API_FINANCIAL`,
`SKOS_MASTER` ou `SKOS_ANALYST`.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `role` | objeto | sim | `{ "name": "…" }` — papel do vínculo: um dos seis de `POST /up/account-role-org`. |
| `org.name` | string | sim | Organização do vínculo. |
| `org.owner` | string | sim | Dono da organização. |
| `accountStatus` | string | sim | `SUSPENDED` ou `CANCELED`. |

**Resposta:** `200` (sem corpo) · `400` — campo ausente, papel ou `accountStatus` fora da lista, ou vínculo
que já está `SUSPENDED` ou `CANCELED` (a própria conta não o reativa).

## PUT /up/account-role-org/status/by-master
Muda o status do vínculo **de outra conta**. **Quem chama:** token com papel `API_MASTER`, `API_ENGINEER`,
`API_ANALYST`, `API_FINANCIAL` ou `SKOS_MASTER`; e ainda tem de ser quem **gere** o papel do vínculo
([Quem gere cada papel](#quem-gere-cada-papel)).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `account` | objeto | sim | `{ "username": "…" }` — conta do vínculo. |
| `role` | objeto | sim | `{ "name": "…" }` — papel do vínculo: um dos seis de `POST /up/account-role-org`. |
| `org.name` | string | sim | Organização do vínculo. |
| `org.owner` | string | sim | Dono da organização. Só é lido quando o papel é `SKOS_ANALYST`; nos demais a organização é procurada entre as de quem chama. |
| `accountStatus` | string | sim | `ACTIVE`, `SUSPENDED` ou `CANCELED`. |
| `orgStatus` | string | sim | `ACTIVE`, `SUSPENDED` ou `CANCELED`. |

**Resposta:** `200` (sem corpo) · `400` — campo ausente, valor fora da lista, ou `"associação indicada não
foi localizada."` · `403` — quem chama não tem `SKOS_MASTER` ativo na organização e o vínculo é
`SKOS_ANALYST`.

## PUT /up/account-role-org/replace-account
Substitui **a conta** de um vínculo existente. **Papel:** administrador na org.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `org` | objeto | sim | `{ "name": "…" }` — org do vínculo. |
| `accountRoleOrg` | objeto | sim | Vínculo atual a alterar. |
| `account` | objeto | sim | Nova conta (`{username}`). |

**Resposta:** `200` quando substituiu · `404` se o vínculo, a conta substituta ou a org não existir — a
mensagem diz **qual dos três** faltou e que **a conta não foi substituída**.

## PUT /up/account-role-org/replace-master-account
Substitui a **conta administradora** do vínculo. **Papel:** administrador na org. Corpo igual ao
`replace-account`.

**Resposta:** `200` quando substituiu · `404` — `"… não existe: a conta master não foi substituída."`

> ⚠️ Até 2026-09-05 estes três `replace-*` respondiam **`204`** quando o vínculo, a conta ou a org não
> existia. `204` é sucesso sem corpo: **trocar a conta administradora de uma organização virava no-op
> silencioso**. Ver [../erros.md](../erros.md).

## PUT /up/account-role-org/replace-role
Substitui o **papel** do vínculo. **Papel:** administrador na org.

**Corpo** (JSON): `org` `{name}` + `accountRoleOrg` (vínculo atual) + `role` `{name}` (novo papel).

**Resposta:** `200` quando substituiu · `404` — `"… não existe: o papel não foi substituído."`

## DELETE /up/account-role-org/by/org/{orgName}
Desassocia a conta da org. **Papel:** administrador na org.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização da qual desassociar |

**Resposta:** `200` (sem corpo).
