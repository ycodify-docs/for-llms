# persistence-q · endpoints · consulta

> Consulta projeções por critério. Guia: [../README.md](../README.md).

## Requisição

`POST /`

Cabeçalhos: `Authorization`, `X-Tenant-Id` (cabeçalho de tenant), `Content-Type: application/json`.

O corpo tem **dois modos**, detectados pelo primeiro caractere:

### Modo array (múltiplos critérios) — primeiro caractere `[`

```json
[
  { "<rótulo_1>": { "<predicado>": "<valor>" } },
  { "<rótulo_2>": { "<predicado>": "<valor>", "<predicado2>": "<valor2>" } }
]
```

### Modo object (critério único) — primeiro caractere `{`

```json
{ "<rótulo>": { "<predicado>": "<valor>" } }
```

### Regras
- Cada item do array tem **exatamente um** par no nível raiz: o **rótulo** → objeto de **predicados**.
- ⚠️ **O rótulo NÃO é livre**: DEVE ser o **nome da projeção (entidade) provisionada** para o tenant
  (ex.: `pedido`) — é a projeção sobre a qual a consulta roda. Um rótulo arbitrário **falha** (`510`,
  projeção de nome `<rótulo>` inexistente). O rótulo também nomeia a chave correspondente na resposta.
- Cada predicado é `{ "<atributo>": "<valor>" }` (igualdade) ou `{ "<atributo>": { "<op>": "<valor>" } }`
  (operadores: `eq/neq/gt/gte/lt/lte/like/ilike/in`).
- Múltiplos predicados combinam por **`_connective`** (padrão **AND**, ou **OR**).
- Há **controles** por critério: `_paging`, `_sorting`, `_count`, `_cache`, associações. Detalhe e
  operadores em **[query-controls.md](../query-controls.md)**.
- O **vocabulário** de atributos é o do modelo provisionado para o tenant; nome desconhecido → erro.
- Como o rótulo é o **nome da projeção**, consultar projeções distintas já dá rótulos únicos; para
  múltiplos critérios sobre a **mesma** projeção, envie um critério por requisição (rótulos repetiriam o
  nome e não se distinguiriam).

## Resposta

Sempre um **array no topo** (mesmo no modo object), na **mesma ordem** dos critérios. Cada item é o
rótulo → lista de registros encontrados:

```json
[
  { "<rótulo_1>": [ { "...": "..." }, { "...": "..." } ] },
  { "<rótulo_2>": [] }
]
```

Garantias da resposta:
- **Ordem preservada** — o i-ésimo item da resposta corresponde ao i-ésimo critério da requisição.
- **Itens vazios aparecem como `[]`** — num `200`, um critério sem resultados ainda ocupa sua posição
  com lista vazia (o cliente deve tratar resultados mistos: alguns preenchidos, outros vazios).
- `200` — há **ao menos um** critério com resultado.
- `204` — **todos** os critérios retornaram vazio.

## Erros
`400` (JSON inválido), `401`/`403` (autorização), `510` (falha de execução da consulta).
Catálogo: [../erros.md](../erros.md).
