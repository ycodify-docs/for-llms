# persistence-q · endpoints · consulta

> Consulta projeções por critério. Guia: [../README.md](../README.md).

## Requisição

`POST /`

Cabeçalhos: `Authorization`, `X-Tenant-Id` (cabeçalho de tenant), `Content-Type: application/json`.

> **URL externa — o segmento `t` escolhe o ambiente.** O path acima é **downstream**; via gateway:
> **produção** `POST /v3/persistence/q/` · **teste** `POST /v3/persistence/t/q/`. Corpo, cabeçalhos e
> respostas são **iguais** nos dois — só muda o prefixo. Ver [autenticação](../../06-autenticacao.md).

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
- Cada item do array tem **um rótulo** no nível raiz (nome da projeção → objeto de **predicados**),
  **mais** os controles opcionais (`_paging`/`_sorting`/`_count`/`_connective`/`_cache`) como **irmãos** do rótulo.
- ⚠️ **O rótulo NÃO é livre**: DEVE ser o **nome da projeção (entidade) provisionada** para o tenant
  (ex.: `pedido`) — é a projeção sobre a qual a consulta roda. Um rótulo arbitrário **falha** com `510` e
  a mensagem `Entidade '<rótulo>' não existe no modelo do tenant '<tenant-id>'. Na consulta, o rótulo deve
  ser o nome da entidade/projeção provisionada para o tenant; confira o nome ou republique o modelo.`
  O rótulo também nomeia a chave correspondente na resposta.
- Cada predicado é `{ "<atributo>": "<valor>" }` (igualdade) ou `{ "<atributo>": { "<op>": "<valor>" } }`
  (operadores: `eq/neq/gt/gte/lt/lte/like/ilike/in`).
- Múltiplos predicados combinam por **`_connective`** no **nível raiz** (irmão do rótulo): **`AND`**
  (padrão) ou **`OR`** (maiúsculas), global ao critério.
- **Controles** ficam no **nível raiz** (irmãos do rótulo, **fora** do objeto de predicados): `_paging`,
  `_sorting` (indexado por `"0"`), `_count`, `_connective`, `_cache`. Já os de **associação**
  (`_associations`/`_populating`/`_level`/`_as`) ficam **dentro** do rótulo. Detalhe, operadores e
  exemplos (inclusive `OR`) em **[query-controls.md](../query-controls.md)**.
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
