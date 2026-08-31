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
- GET /up/account/by/org/{orgName} — conta do próprio chamador (NÃO lista a org)
- GET /up/account/by/org/{orgName}/org-owner/{orgOwner} — **listar os membros da org**
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
⚠️ **Apesar do nome, NÃO lista os membros da organização.** O dono da org é deduzido do token e o
resultado é filtrado pela conta de quem chama: devolve **apenas o próprio chamador**. Para os membros,
use a rota com `org-owner`, abaixo. **Papéis:** administrador / engenheiro / analista (na org) — os
demais recebem `403`, mesmo o resultado sendo a própria conta.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |

**Resposta:** `200` — array com **uma** conta, a do portador do token (mesma forma da rota abaixo);
`204` sem corpo se não houver vínculo. Sem `Authorization` → `401`.

## GET /up/account/by/org/{orgName}/org-owner/{orgOwner}
**Esta é a rota de listagem:** devolve os **membros** da organização, cada conta com os seus **vínculos**
(papel e status). **Papéis:** administrador / engenheiro / analista **na própria organização**; os demais
recebem `403`. Sem `Authorization` → `401`.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |
| `orgOwner` | dono da organização |

**Resposta:** `200` — array de contas. Por conta: `username`, `name`, `email`, `status` e
`accountRoleOrgs[]`; a senha **nunca** é devolvida. Cada item de `accountRoleOrgs` traz `role.name`,
`org` (`name` e `owner`), `accountStatus` e `orgStatus`. Sem membro → `204` **sem corpo** (não é `[]`).

> ⚠️ **A lista mistura vínculos ativos e suspensos — e nada os distingue à vista.** O serviço descarta
> somente o que estiver **cancelado**; vínculos **suspensos** e contas **pendentes** vêm na resposta.
> Além disso, `status` é o estado **global** da conta e **não diz nada sobre esta organização**: a mesma
> pessoa pode estar suspensa aqui e ativa em outra org. Considere alguém **membro ativo nesta
> organização** apenas quando `status`, `accountStatus` e `orgStatus` forem **todos** `ACTIVE` — o filtro
> é responsabilidade do consumidor.

> Não há rota autenticada que resolva **uma** conta por `username` (só a verificação de existência, sem
> dados, e a conta do próprio token). Para saber se alguém pertence a uma org, liste por esta rota e
> procure — aplicando o filtro de status acima.

## POST /up/account/e-mail/send
Envia uma notificação por e-mail (usuário autenticado).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `to` | string | sim | Destinatário. |
| `subject` | string | sim | Assunto. |
| `body` | string | sim | Corpo (texto). |

**Resposta:** `200` — `{}`.
