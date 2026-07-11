# orgid · `/up` · contrato

> ⚠️ **IGNORAR TEMPORARIAMENTE.** A API de contratos **deve ser desconsiderada por ora** — não a use
> nas integrações nesta fase. Documentada apenas para referência; aguarde liberação antes de adotá-la.

> Domínio **plataforma** (`/up`). Contrato/assinatura de uma **organização** (plano, validade, status).
> Convenção: **um contrato ativo** por org. Guia: [../README.md](../README.md).
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. Leituras
> conferem **posse** da org. Erros: [../erros.md](../erros.md).

## Contents
- POST /up/contract — criar
- PUT /up/contract — atualizar
- GET /up/contract/by/uuid/{uuid}/exists — existe
- GET /up/contract/by/uuid/{uuid}/org/{orgName} — ler por uuid+org
- GET /up/contrac/org/{orgName} — listar por org

---

## POST /up/contract
Cria o contrato da org. **Papel:** administrador (na org).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `orgName` | string | sim | Organização do contrato. |
| `plan` | objeto | não | `{ "id": <1..4> }` — plano (1 básico, 2 padrão [default], 3 pro, 4 enterprise). |
| `ttl` | inteiro | só se `plan.id`=4 | Tempo de vida (enterprise). |
| `threshold` | inteiro | só se `plan.id`=4 | Limite (enterprise). |

**Resposta:** `200` — contrato criado: `uuid`, `status:"ACTIVE"`, `hiredin`, `expiresin` (≈1 ano), `plan.id`.

## PUT /up/contract
Atualiza o contrato. **Papéis:** administrador / financeiro (na org).

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `uuid` | string | sim | Contrato a atualizar. |
| `orgName` | string | sim | Org do contrato (deve casar). |
| `status` | string | não | Novo status (`ACTIVE`/`FINISHED`). |
| `expiresin` | inteiro | não | Novo vencimento (timestamp). |

**Resposta:** `200` (sem corpo).

## GET /up/contract/by/uuid/{uuid}/exists
Verifica existência. **Papéis:** administrador / financeiro.

| Path-var | Significado |
|---|---|
| `uuid` | identificador do contrato |

**Resposta:** `200` (existe) / `204` (não).

## GET /up/contract/by/uuid/{uuid}/org/{orgName}
Lê o contrato. **Papéis:** administrador / financeiro (posse da org).

| Path-var | Significado |
|---|---|
| `uuid` | identificador do contrato |
| `orgName` | organização do contrato |

**Resposta:** `200` — contrato (`id`, `uuid`, `orgName`, `status`, `hiredin`, `expiresin`, `plan`, `ttl?`, `threshold?`); `204` se não encontrado.

## GET /up/contrac/org/{orgName}
Lista contratos da org. **Papéis:** administrador / financeiro (posse).

> ⚠️ O caminho é **`/up/contrac/org/...`** (sem o "t" em "contrac") — use **exatamente** assim; é o contrato real do endpoint.

| Path-var | Significado |
|---|---|
| `orgName` | organização |

**Resposta:** `200` — array de contratos; `204` se vazio.
