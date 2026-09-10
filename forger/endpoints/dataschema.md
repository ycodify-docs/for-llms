# forger · endpoints · dataschema

> **dataschema** = esquema vinculado a um **project** e a um **database**. É a **âncora do isolamento
> por tenant**: sua criação gera o `tenant-id` do sistema e materializa o esquema físico no banco de
> leitura. Por convenção padrão, há **um dataschema por bounded context** dentro de um mesmo database
> ([topologia padrão](../../02-conceitos.md#convenção-padrão-de-topologia-project--bounded-context--esquema)).
> Guia: [../README.md](../README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Criar

`POST /org/{org}/project/{project}/database/{databaseId}/dataschema`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | sim | Nome do esquema. Minúsculas, dígitos e `_`, começando por letra (`^[a-z][a-z0-9_]*$`), **máx. 12 caracteres** — o limite é do esquema físico, e não o mesmo do `project`, que aceita 16. |
| `alias` | string | não | Apelido. |
| `description` | string | não | Descrição. |
| `dbsqlminimumconnidle` | inteiro | não | Mínimo de conexões ociosas no pool de leitura. |
| `dbsqlmaximumpoolsize` | inteiro | não | Tamanho máximo do pool de leitura. |

Efeito: gera o **`tenant-id`** (UUID) e cria o **esquema físico** no banco de leitura.
Resposta `201`: `{ "id": <número>, "message": "..." }` (o `tenant-id` fica associado ao esquema).
Erros: `400`, `403`, `500`.

## Ler / Listar

- `GET /org/{org}/project/{project}/database/{databaseId}/dataschema/{dataSchema}` → `200`/`204`.
- `GET /org/{org}/project/{project}/database/{databaseId}/dataschema` → `200` array; `204` vazio.

## Atualizar

`PUT .../dataschema/{dataSchema}` — corpo: `logversion` + editáveis (`alias`, `description`, `status`,
`dbsqlminimumconnidle`, `dbsqlmaximumpoolsize`). `200`; conflito → `409`.

> **Campo `status` — gate operacional (`MODELING` ↔ `RUNNING`).** O `status` do dataschema **não** é um
> rótulo passivo: ele **controla** o que pode ser feito sobre o esquema e sobre o tenant. Defina/transite
> o `status` pelo `PUT` acima.
>
> | `status` | Schema (entity) | Interpretação/execução do modelo |
> |---|---|---|
> | **`MODELING`** | **editável** — criar/alterar/remover `entity` (e atributos/associações) é permitido | **não ocorre** — persistence-crs e persistence-q **não operam** sobre este tenant |
> | **`RUNNING`** | **congelado** — o **forger rejeita** criar/alterar `entity` neste estado | **ocorre** — persistence-crs/persistence-q interpretam e executam o modelo |
>
> **Regra para o agente:** para **editar schema** (criar/alterar entity), o dataschema precisa estar em
> `MODELING`; para **operar** (comandos/consultas via persistence-crs/persistence-q), em `RUNNING`.

### Alterar o schema de um sistema em operação

A transição de `status` **não é só um rótulo no banco**: ela remove e republica os modelos que o motor
lê. Por isso alterar um schema em operação não é "editar e voltar" — o passo **4** abaixo é o que
costuma faltar, e sem ele a volta a `RUNNING` é recusada.

**1. Descubra a `logversion` corrente** (o `PUT` a exige, e uma versão errada responde `409`):

```bash
curl -H "Authorization: Bearer $TOKEN" \
  ".../org/acme/project/vendas/database/7/dataschema/pedidos"
# → { "id": 12, "name": "pedidos", "status": "RUNNING", "logversion": 4, ... }
```

**2. Vá para `MODELING`** — o tenant para de operar:

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  ".../org/acme/project/vendas/database/7/dataschema/pedidos" \
  -d '{"logversion": 4, "status": "MODELING"}'
```

Aqui **as duas chaves do modelo são removidas**: a spec de entidades e o `.model.json`. A partir deste
instante o motor recusa **consulta e comando** com `510` — de propósito, porque o tenant está
declaradamente em remodelagem.

**3. Edite as entities** (só é permitido em `MODELING`) — ver [entity.md](entity.md).

**4. Republique o `.model.json`** — este é o passo que falta em quase todo roteiro escrito à mão:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  ".../org/acme/project/vendas/tenant/$TENANT_ID/model" \
  -F "file=@pedidos.model.json"
```

**5. Volte para `RUNNING`** (a `logversion` avançou no passo 2 — releia-a):

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  ".../org/acme/project/vendas/database/7/dataschema/pedidos" \
  -d '{"logversion": 5, "status": "RUNNING"}'
```

A spec de entidades volta sozinha. O `.model.json` **não** — e é por isso que o passo 4 existe.

> **Se você pular o passo 4**, o `PUT` do passo 5 é recusado:
>
> ```json
> 400: DataSchema 'pedidos' não pode voltar a RUNNING: o modelo de escrita não está publicado.
> ```
>
> **Nada é gravado** e o dataschema **continua em `MODELING`** — repita o passo 4 e depois o 5.

**Por que só um dos dois modelos volta sozinho.** A spec de entidades a plataforma sabe remontar, porque
ela deriva das entities, que vivem no banco. O `.model.json` é **artefato seu**: a plataforma não guarda
outra cópia dele, e quem o repõe é quem o tem.

**Não invalide cache manualmente em nenhum ponto.** Remover e publicar é da plataforma, e nenhum dos dois
modelos expira por tempo — o que os tira do cache é remoção, nunca prazo.

## Remover

`DELETE .../dataschema/{dataSchema}` → `200`/`204`. **Bloqueado** enquanto o esquema físico existir
(há projeções/conteúdo). Ver também a remoção em nível de projeção em [entity.md](entity.md).

## Ciclo de vida
Metadados + efeito físico (criação de esquema) + geração do `tenant-id`. A partir daqui o sistema tem
identidade de tenant para comandos/eventos/consultas.

## Coordenação
- **CP-1/CP-2:** o `tenant-id` deste esquema é a chave usada por persistence-crs, es-n e persistence-q;
  as projeções (entities) deste esquema serão consultadas por persistence-q.
Habilita criar **entity** ([entity.md](entity.md)) e publicar **model** ([model.md](model.md)).
