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
| `name` | string | sim | Nome do esquema. Minúsculas, dígitos e `_`, começando por letra (`^[a-z][a-z0-9_]*$`), **máx. 16 caracteres**. |
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

> **Sempre leia antes de atualizar.** `logversion` é **versão otimista**: precisa ser o valor **atual**
> do recurso, obtido no `GET` acima. Chutar `0` (ou reusar um valor antigo) responde **`409`** e **nada
> acontece** — nem a gravação, nem o efeito no cache descrito abaixo. Receita: `GET` → pegue
> `logversion` → `PUT` com esse valor.
>
> **`status` inválido não é rejeitado.** O campo **não** é validado contra a lista de valores: um erro
> de digitação (`RUNNIG`, `RUNING`) é **gravado como veio** e responde `200`. Como a transição é
> comparada contra `MODELING`/`RUNNING`, o efeito é **falha silenciosa**: nada é publicado nem removido
> do cache, e o dataschema fica num status que **não** libera edição de entity (só `MODELING` libera)
> **nem** habilita operação. Confira a grafia; para sair do estado, faça outro `PUT` com o valor certo.

> **Campo `status` — gate operacional (`MODELING` ↔ `RUNNING`).** O `status` do dataschema **não** é um
> rótulo passivo: ele **controla** o que pode ser feito sobre o esquema e sobre o tenant, e **é o gatilho
> da publicação do modelo de entidades**. Defina/transite o `status` pelo `PUT` acima.
>
> | `status` | Schema (entity) | Modelo de entidades no cache | Interpretação/execução do modelo |
> |---|---|---|---|
> | **`MODELING`** | **editável** — criar/alterar/remover `entity` (e atributos/associações) é permitido | **ausente** | **não ocorre** — persistence-crs e persistence-q **não operam** sobre este tenant |
> | **`RUNNING`** | **congelado** — o **forger rejeita** criar/alterar `entity` neste estado | **publicado** | **ocorre** — persistence-crs/persistence-q interpretam e executam o modelo |
>
> O valor é normalizado (`trim` + maiúsculas) antes de ser gravado e comparado: `running` e `RUNNING`
> são a mesma transição.
>
> **Regra para o agente:** para **editar schema** (criar/alterar entity), o dataschema precisa estar em
> `MODELING`; para **operar** (comandos/consultas via persistence-crs/persistence-q), em `RUNNING`. Para
> alterar o schema de um sistema já em operação: transite `RUNNING → MODELING` (via `PUT` no `status`),
> edite as entities, e volte a `MODELING → RUNNING`.

### Efeito da transição sobre o cache distribuído

A transição de `status` **publica e despublica o modelo de entidades** (a descrição das entidades que
correspondem às projeções dos agregados) no cache distribuído, sob a referência do `tenant-id`:

| Transição | Efeito |
|---|---|
| `MODELING → RUNNING` | **publica** o estado atual das entidades, **sem prazo de validade** e **sobrescrevendo** o valor anterior |
| `RUNNING → MODELING` | **remove** a publicação |
| qualquer outra (ou nenhuma mudança de `status`) | nenhum efeito no cache |

O forger é o **único publicador** dessa referência — os consumidores (persistence-crs, persistence-q)
apenas leem. Como não há expiração, vale o que foi publicado na última transição para `RUNNING`, até a
próxima transição.

**Atomicidade:** se a operação no cache falhar, o `status` é **revertido** no banco e a requisição
responde `500` — nada fica em estado intermediário (nunca `RUNNING` sem modelo publicado, nem
`MODELING` com modelo publicado). Repita o `PUT` relendo o `logversion` atual.

**Consequência prática:** editar entities e voltar a `RUNNING` **republica** o modelo, e o efeito é
imediato para os consumidores. Um tenant que nunca passou por `MODELING → RUNNING` **não tem** modelo
de entidades publicado, e consultas sobre ele falham no consumidor.

## Remover

`DELETE .../dataschema/{dataSchema}` → `200`/`204`. **Bloqueado** enquanto o esquema físico existir
(há projeções/conteúdo). Ver também a remoção em nível de projeção em [entity.md](entity.md).

A remoção também **despublica o modelo de entidades** do cache distribuído — a publicação não tem
prazo de validade, então ficaria órfã apontando para um tenant inexistente. A despublicação acontece
**antes** de o registro ser apagado: se o cache falhar, **nada é removido** e a requisição responde
`500`, podendo ser repetida. Dataschema que nunca foi publicado é removido normalmente.

## Ciclo de vida
Metadados + efeito físico (criação de esquema) + geração do `tenant-id`. A partir daqui o sistema tem
identidade de tenant para comandos/eventos/consultas.

## Coordenação
- **CP-1/CP-2:** o `tenant-id` deste esquema é a chave usada por persistence-crs, es-n e persistence-q;
  as projeções (entities) deste esquema serão consultadas por persistence-q.
Habilita criar **entity** ([entity.md](entity.md)) e publicar **model** ([model.md](model.md)).
