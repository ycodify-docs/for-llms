# orgid · `/ua` · papel

> Domínio **externo** (`/ua`). Papéis com **dono** (`owner`, espaço de nomes) e visibilidade
> (`ispublic`). A associação **conta-papel** está em [ua-associacao.md](ua-associacao.md).
> Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório (exceto a listagem pública); POST/PUT enviam
> `Content-Type: application/json`. **Papéis aceitos:** administrador / engenheiro (no `owner` do papel).
> Erros: [../erros.md](../erros.md).

## Contents
- POST /ua/role — criar papel
- PUT /ua/role — atualizar papel
- GET /ua/role/by/name/{roleName}/owner/{roleOwner} — ler papel
- GET /ua/role/by/owner/{roleOwner} — listar papéis do dono
- GET /ua/open/role/owner/{roleOwner} — **público:** listar papéis públicos
- DELETE /ua/role/by/name/{roleName}/owner/{roleOwner} — remover papel

---

## POST /ua/role
Cria um papel. **Papéis:** administrador / engenheiro (no `owner`).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `name` | string | sim | Nome do papel. |
| `owner` | string | sim | Dono (espaço de nomes) do papel. |
| `label` | string | não | Rótulo de exibição. |
| `status` | string | não | Situação (padrão `ACTIVE`). |
| `ispublic` | boolean | não | Se aparece na listagem pública (padrão `false`). |

**Resposta:** `200` — `{ "id": <number> }`.

## PUT /ua/role
Atualiza um papel (mesmos campos do POST). **Resposta:** `200` (sem corpo).

## GET /ua/role/by/name/{roleName}/owner/{roleOwner}
Lê um papel.

| Path-var | Significado |
|---|---|
| `roleName` | nome do papel |
| `roleOwner` | dono do papel |

**Resposta:** `200` — papel (`id`, `name`, `owner`, `label`, `status`, `ispublic`); `400` se inválido.

## GET /ua/role/by/owner/{roleOwner}
Lista os papéis de um dono.

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel |

**Resposta:** `200` — array de papéis.

## GET /ua/open/role/owner/{roleOwner}
**Público (sem auth).** Lista **apenas** papéis com `ispublic=true`.

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel |

**Resposta:** `200` — array de `{ name, label }` (só públicos).

## DELETE /ua/role/by/name/{roleName}/owner/{roleOwner}
Remove um papel.

| Path-var | Significado |
|---|---|
| `roleName` | nome do papel |
| `roleOwner` | dono do papel |

**Resposta:** `200` (sem corpo).
