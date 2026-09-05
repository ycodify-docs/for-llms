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

> **Campos de `valueObject`:** a forma do valor segue a forma **declarada** no modelo — grupo de campos
> recebe objeto (`single`) ou array de objetos (`multiple`); atributo tipado direto recebe escalar
> (`single`) ou **array de escalares** (`multiple`, ex.: `"diassemana": ["terca"]`). Forma divergente →
> `400` dizendo a forma esperada. Ver
> [spec/model-format.md § data.valueObject](../spec/model-format.md).

> **Campo `status` (obrigatório, exceto na criação):** todo comando envia `status` = **estado atual**
> do agregado (lido do banco de escrita), **nunca** o estado pretendido após o comando (`endState`). O
> **comando de criação** é a **única exceção** — não envia `status`. Divergência com o estado real →
> `510` (alguém avançou o agregado antes; reenvie sobre o estado atualizado). Ver
> [README — regra do status](../README.md#estados-transições-e-concorrência).

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
`400` (corpo inválido, incluindo `valueObject` na forma errada), `403` (tenant não autorizado **ou
usuário sem o papel exigido pelo comando**), `510` (falha de transição/processamento e demais exceções).
Catálogo: [../erros.md](../erros.md).
