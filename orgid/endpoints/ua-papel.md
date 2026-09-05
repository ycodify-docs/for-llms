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
Atualiza um papel.

> ⚠️ **`id` é obrigatório** — e **não** vem do POST. O corpo do POST não tem `id`; o update
> localiza o papel por ele. Descubra o `id` com `GET /ua/role/by/name/{roleName}/owner/{roleOwner}`
> antes de atualizar.

**Corpo** (JSON): os campos do POST **mais** o `id`:

```json
{ "id": 101, "name": "ASSOCIADO", "owner": "acme", "label": "Associado", "status": "ACTIVE", "ispublic": true }
```

**Resposta:** `200` (sem corpo) quando gravou · `400` se faltar o `id` · `404` se o papel não existe
ou pertence a outro `owner`.

> Até 2026-09-05 um `PUT` sem `id` respondia **`204`** e não gravava nada. `204` é família 2xx —
> sucesso sem corpo —, então a mensagem de erro sumia no protocolo e o chamador seguia achando que
> tinha configurado. Se você vir `204` aqui, está falando com uma versão antiga do orgid.

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

> ⚠️ **É o único público do orgid que NÃO começa por `/open/`** — os segmentos estão invertidos
> (`/ua/open/…` em vez de `/open/ua/…`). Isso não é cosmético: as regras de autorização da plataforma são
> escritas sobre o prefixo `/open/**`, e **este path escapa delas** e cai na regra de autenticado. No
> `OrgIdSecurityConfig` do orgid standalone continua assim — liberados só `/open/**`, `/z/**`,
> `/*/unsecured/**`, swagger e actuator —, então **o orgid sozinho responde `401` aqui, apesar do
> "público" acima**. O que está implantado é o `composer`, que desde 2026-09-05 libera os **dois**
> prefixos e faz o endpoint honrar o contrato documentado.
>
> Ao escrever gateway, proxy ou nova cadeia de segurança: **público no orgid = dois prefixos**,
> `/open/**` **e** `/ua/open/**`.

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel |

**Resposta:** `200` — array de `{ name, label }` (só públicos). O filtro é do próprio endpoint: papel com
`ispublic=false` não aparece, e de cada papel saem **só** `name` e `label` — nunca `id`, `status` ou
`owner`. É o que torna seguro expô-lo sem token.

## DELETE /ua/role/by/name/{roleName}/owner/{roleOwner}
Remove um papel.

| Path-var | Significado |
|---|---|
| `roleName` | nome do papel |
| `roleOwner` | dono do papel |

**Resposta:** `200` (sem corpo) quando removeu · `404` — `"O papel informado não existe: nada foi
removido."`

> ⚠️ Até 2026-09-05 este `DELETE` respondia **`204`** quando o papel não existia. Num `DELETE`, `204` é o
> código **canônico de sucesso** — a leitura era a oposta da correta, e a remoção que não aconteceu passava
> por feita. Ver [../erros.md](../erros.md).
