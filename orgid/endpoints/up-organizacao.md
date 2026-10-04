# orgid · `/up` · organização

> Domínio **plataforma** (`/up`). CRUD de **organização**. O `name` da org é o `{org}` usado nos
> caminhos do forger. Guia: [../README.md](../README.md).
>
> **Comum a todos os endpoints abaixo:** cabeçalho `Authorization` (token portador) obrigatório;
> POST/PUT enviam `Content-Type: application/json`. Autorização **escopada na organização** (o portador
> precisa do papel **naquela** org). Erros: [../erros.md](../erros.md).

## Contents
- POST /up/org — criar
- PUT /up/org — atualizar
- GET /up/org/by/name/{orgName} — ler
- GET /up/org/by/name/{orgName}/owner/{orgOwner} — ler por nome+dono
- GET /up/org/by/name/{orgName}/owner/{orgOwner}/exists — existe
- GET /up/org/by/owner/{orgOwner} — listar do dono
- DELETE /up/org/by/org/{orgName}/org-owner/{orgOwner} — remover

---

## POST /up/org
Cria uma organização. **Papel:** administrador.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `name` | string | sim | Nome da org (vira o `{org}` do forger; único). |
| `client` | string | **sim** | **Nome da empresa cliente** que contratou a plataforma — toda organização deve sinalizar o seu. **Máx. 12 caracteres.** Ausente/em branco → `400`; acima de 12 → `400`. |
| `alias` | string | não | Apelido legível. |
| `email` | string | não | Contato da org. |
| `status` | string | não | Situação (padrão `ACTIVE`). |
| `cardIdentifier` | string | não | Identificador de pagamento. |

> O `owner` é definido automaticamente como o usuário autenticado (não enviar).

**Resposta:** `201`/`200` — a org criada com `id`. Erros: `400`, `403`.

## PUT /up/org
Atualiza a organização. **Papel:** administrador **na org** (verificado pelo `name`).

**Corpo** (JSON): `name` (string, **obrigatório** — identifica a org) + campos editáveis
(`alias`, `email`, `status`, `cardIdentifier`). O `client` é definido na **criação** (obrigatório) — não
faz parte dos campos alteráveis por aqui.

**Resposta:** `200` (sem corpo) quando gravou · `403` se não for administrador da org · `404` —
`"Organização não localizada: nenhum dado foi atualizado."`

## GET /up/org/by/name/{orgName}
Lê a org pelo nome. **Papéis:** administrador / engenheiro / analista / financeiro.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |

**Resposta:** `200` com o objeto org (`id`, `owner`, `name`, `alias`, `status`, `email`, …); `204` se não existir.

## GET /up/org/by/name/{orgName}/owner/{orgOwner}
Lê a org por nome **e** dono. **Papéis:** administrador / engenheiro / analista / financeiro.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |
| `orgOwner` | dono (username) da organização |

**Resposta:** `200` org; `204` se não existir.

## GET /up/org/by/name/{orgName}/owner/{orgOwner}/exists
Verifica existência. **Papéis:** administrador / engenheiro / analista / financeiro.

| Path-var | Significado |
|---|---|
| `orgName` | nome da organização |
| `orgOwner` | dono da organização |

**Resposta:** `200` (existe) · `204` (não existe). Sem corpo.

## GET /up/org/by/owner/{orgOwner}
Lista as orgs de um dono. **Papéis:** administrador / engenheiro (a posse precisa casar com `orgOwner`).

| Path-var | Significado |
|---|---|
| `orgOwner` | dono (username) cujas orgs serão listadas |

**Resposta:** `200` array de orgs; `204` se vazio.

## DELETE /up/org/by/org/{orgName}/org-owner/{orgOwner}
Remove a org. **Papel:** administrador **na org**.

| Path-var | Significado |
|---|---|
| `orgName` | nome da org a remover |
| `orgOwner` | dono da org |

**O que é removido:** a organização e os vínculos conta-papel-organização de **todos** os membros, em
todos os papéis. Não tem volta. **Não** são removidos: as contas dos membros, e os papéis e contas de
aplicação (`/ua`) cujo dono é a organização — remova-os pelas rotas de `/ua`, vínculo antes do papel.

**Pré-condição: a organização não pode ter recurso no forger** — nenhum projeto e nenhuma conexão de
banco. Havendo qualquer um dos dois, a resposta é `400` e **nada é removido**. Desmonte antes, na ordem
de [forger — dataschema, remover](../../forger/endpoints/dataschema.md#remover), depois o projeto e a
conexão. **Remover a organização é o último passo.**

**Resposta:** `200` (sem corpo). `400` se a organização ainda tem projeto ou conexão de banco no forger.
`403` se não for administrador. Se não existir organização com esse nome **para esse dono**, a resposta
também é `200` e nada é removido — confira `orgOwner`.

> Em produção desde 2026-10-02T22:11Z. Até então esta rota respondia `403` a **qualquer** chamada,
> inclusive do administrador da organização, e nenhuma organização era removida.
