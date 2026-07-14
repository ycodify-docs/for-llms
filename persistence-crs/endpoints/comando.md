# persistence-crs · endpoints · comando

> Executa um **comando** sobre um agregado. Guia: [../README.md](../README.md).

## Requisição

`POST /a/{boundedContext}/{aggregateType}`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id` (cabeçalho de tenant), `Content-Type: application/json` |
| `{boundedContext}` | contexto delimitado do agregado |
| `{aggregateType}` | tipo do agregado |

Corpo: o **comando** identificado pelo seu nome, com os dados:

```json
{ "<nomeDoComando>": { "<campo>": "<valor>", "...": "..." } }
```

O nome do comando, seus campos e as transições válidas são definidos no **model** publicado para o
tenant (ver [forger/model](../../forger/endpoints/model.md) e [examples/](../../examples/README.md)).

> **Campo `status` (obrigatório, exceto na criação):** todo comando envia `status` = **estado atual**
> do agregado (lido do banco de escrita), **nunca** o estado pretendido após o comando (`endState`). O
> **comando de criação** é a **única exceção** — não envia `status`. Divergência com o estado real →
> `510` (alguém avançou o agregado antes; reenvie sobre o estado atualizado). Ver
> [README — regra do status](../README.md#estados-transições-e-concorrência).

### Transição sobre um agregado existente

Mesma rota da criação — `POST /a/{boundedContext}/{aggregateType}` — **sem** `/{uuid}` no path. O agregado
alvo é apontado pelo campo **`id` dentro do corpo** do comando, valorado com o **UUID do registro do
agregado** (o `id` devolvido na criação / o `aggregateid` da projeção; **não** a PK numérica da projeção):

```json
{ "<nomeDoComando>": { "id": "<uuid-do-agregado>", "status": "<estado atual>", "<campo>": "<valor>" } }
```

Só a **criação** não envia `id` nem `status` (o agregado ainda não existe). O path `/{uuid}` existe apenas
para **leitura** (`GET .../{uuid}`) e **histórico** (`GET .../{uuid}/history`), não para comandos.

## Resposta

`200`:

```json
{ "id": "<id do agregado>", "status": "<estado resultante>" }
```

> Só `id` e `status` são devolvidos — **não** os dados do comando. A projeção é atualizada de forma
> assíncrona depois (ver [arquitetura](../../01-arquitetura.md)).

## Comportamento

Segue o [ciclo de vida de um comando](../README.md#ciclo-de-vida-de-um-comando): auth/tenant → carga do
estado → validação de transição (`fromState`) → regra/coordenação no br-service (se o modelo exigir) →
validação do estado de destino (`endState`) → gravação do evento (notifica es-n) → resposta.

## Erros
`400` (corpo inválido), `403` (tenant não autorizado), `510` (falha de transição/processamento e demais
exceções). Catálogo: [../erros.md](../erros.md).
