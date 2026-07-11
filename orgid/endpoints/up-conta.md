# orgid · `/up` · conta

> Domínio **plataforma** (`/up`). Conta do usuário da plataforma. **Senha nunca é devolvida.** Registro
> e recuperação são públicos → [publico.md](publico.md). Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`.
> Papéis aceitos (salvo indicação): administrador / engenheiro / analista / financeiro. Erros: [../erros.md](../erros.md).

## Contents
- GET /up/account/by/jwt — conta autenticada
- PUT /up/account — atualizar perfil
- PUT /up/account/password — atualizar senha
- DELETE /up/account/by/jwt — remover a própria conta
- GET /up/account/by/org/{orgName} — listar contas da org
- GET /up/account/by/org/{orgName}/org-owner/{orgOwner} — listar contas da org (com dono)
- POST /up/account/e-mail/send — enviar e-mail

---

## GET /up/account/by/jwt
Retorna a conta do **portador do token** (resolvida pelo token, sem parâmetros).

**Resposta:** `200` — conta (`id`, `username`, `name`, `email`, `status`, endereço, `accountRoleOrgs`;
`password` = nulo); `204` se não encontrada.

## PUT /up/account
Atualiza o **perfil** da própria conta (o `username` é forçado ao do token).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `name` | string | não | Nome. |
| `email` | string | não | E-mail. |
| `endRua`,`endNumero`,`endComplemento`,`endBairro`,`endCidade`,`endUf`,`endPais`,`endCep` | string | não | Endereço. |

**Resposta:** `200` (sem corpo).

## PUT /up/account/password
Troca a senha da própria conta.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `password` | string | sim | Nova senha. |
| `oldPassword` | string | não | Senha atual (validação). |

**Resposta:** `200` (sem corpo).

## DELETE /up/account/by/jwt
Remove a conta do portador do token. Sem parâmetros/corpo. **Resposta:** `200`.

## GET /up/account/by/org/{orgName}
Lista as contas de uma org. **Papéis:** administrador / engenheiro / analista (na org).

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |

**Resposta:** `200` array de contas (senha nula); `204` se vazio.

## GET /up/account/by/org/{orgName}/org-owner/{orgOwner}
Igual ao anterior, qualificando o dono.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |
| `orgOwner` | dono da organização |

**Resposta:** `200` array de contas (senha nula); `204` se vazio.

## POST /up/account/e-mail/send
Envia uma notificação por e-mail (usuário autenticado).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `to` | string | sim | Destinatário. |
| `subject` | string | sim | Assunto. |
| `body` | string | sim | Corpo (texto). |

**Resposta:** `200` — `{}`.
