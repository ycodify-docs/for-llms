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

### O que a transição faz com o modelo no cache

> **⚠️ Vigência:** o que esta seção descreve vale **a partir da versão do forger que carregar
> `f45b22c`** (mergeado em `09dbfcb`) — **ainda não implantado em 2026-09-10**. Na superfície que está
> no ar, `RUNNING → MODELING` derruba **apenas** a spec de entidades, e a volta a `RUNNING` **não
> verifica nada**: o passo 3 do roteiro abaixo parece opcional porque hoje ele é. Confirme a versão
> implantada antes de tratar o passo 3 como dispensável — ele deixa de ser.

A transição de `status` **não é só um rótulo no banco**: ela publica e remove os modelos que o motor
consome. Isso muda o roteiro de alterar um schema em operação, e é a parte que costuma ser feita à mão
sem necessidade.

| Transição | O que acontece |
|---|---|
| **`RUNNING → MODELING`** | **as duas chaves do modelo são removidas** — a spec de entidades (read model) e o modelo de escrita (`.model.json`). O motor passa a recusar **consulta e comando** com `510`, e é assim de propósito: o tenant está declaradamente em remodelagem |
| **`MODELING → RUNNING`** | a spec de entidades é **republicada pela plataforma**, remontada da definição que vive no banco. O modelo de escrita **não** é reconstruído — veja o pré-requisito abaixo |

> ⚠️ **Pré-requisito para voltar a `RUNNING`: o `.model.json` tem de estar publicado.** Se não estiver, o
> `PUT` é recusado com `400`, **nada é gravado** e o dataschema **continua em `MODELING`**. Republique-o
> (`POST .../tenant/{tenantId}/model`, aceito com o dataschema em `MODELING`) e repita a transição.
>
> A assimetria tem motivo: a spec de entidades a plataforma sabe remontar, porque ela deriva do que está
> no banco. O `.model.json` é **artefato seu** — a plataforma não guarda outra cópia dele, e quem o repõe
> é quem o tem.

**Roteiro completo, para alterar o schema de um sistema já em operação:**

1. `RUNNING → MODELING` (`PUT` no `status`) — as duas chaves caem, o tenant para de operar;
2. edite as entities;
3. **republique o `.model.json`**;
4. `MODELING → RUNNING` — a spec volta sozinha, e o passo 3 é verificado.

**Não invalide cache manualmente em nenhum ponto.** A publicação e a remoção são da plataforma, e nenhum
dos dois modelos expira por tempo: o que os tira do cache é remoção, nunca prazo.

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
