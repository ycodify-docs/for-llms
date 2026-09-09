# persistence-q · endpoints · consulta

> Consulta projeções por critério. Guia: [../README.md](../README.md).

## Requisição

`POST /`

Cabeçalhos: `Authorization`, `X-Tenant-Id` (cabeçalho de tenant), `Content-Type: application/json`.

O corpo tem **dois modos**, detectados pelo primeiro caractere:

### Modo array (múltiplos critérios) — primeiro caractere `[`

```json
[
  { "<rótulo_1>": { "<predicado>": "<valor>" }, "_connective": "AND" },
  { "<rótulo_2>": { "<predicado>": "<valor>", "<predicado2>": "<valor2>" }, "_connective": "OR" }
]
```

Os controles ficam ao lado do rótulo, dentro do **mesmo item** do array — cada item tem os seus.

### Modo object (critério único) — primeiro caractere `{`

```json
{
  "<rótulo>": { "<predicado>": "<valor>" },
  "_paging": { "_maxRegisters": 50, "_firstRegister": 0 }
}
```

### Regras
- Cada item do array tem **exatamente um** par de critério no nível raiz: o **rótulo** → objeto de
  **predicados**. As demais chaves daquele nível são os controles, todas com prefixo `_`.
- ⚠️ **O rótulo NÃO é livre**: DEVE ser o **nome da projeção (entidade) provisionada** para o tenant
  (ex.: `pedido`) — é a projeção sobre a qual a consulta roda. Um rótulo arbitrário **falha** (`510`,
  projeção de nome `<rótulo>` inexistente). O rótulo também nomeia a chave correspondente na resposta.
- Cada predicado é `{ "<atributo>": "<valor>" }` (igualdade) ou `{ "<atributo>": { "<op>": "<valor>" } }`
  (operadores: `eq/neq/gt/gte/lt/lte/like/ilike/in`).
- Múltiplos predicados combinam por **`_connective`** (padrão **AND**, ou **OR**).
- Os **controles** — `_paging`, `_sorting`, `_count`, `_connective`, `_cache` — são **irmãos do
  rótulo**, no mesmo nível dele, e **não** vão dentro do objeto de predicados. Controle posto dentro do
  rótulo é **ignorado em silêncio**, sem erro. Detalhe e operadores em
  **[query-controls.md](../query-controls.md)**.
- O **vocabulário** de atributos é o do modelo provisionado para o tenant; nome desconhecido → **`400`**,
  com os nomes recusados **e** a lista dos declarados na mensagem. Vale em cada nível: o mesmo se aplica
  aos atributos de uma associação ou de um componente. Chaves de controle (as que começam com `_`) não
  passam por essa checagem.
- Como o rótulo é o **nome da projeção**, consultar projeções distintas já dá rótulos únicos; para
  múltiplos critérios sobre a **mesma** projeção, envie um critério por requisição (rótulos repetiriam o
  nome e não se distinguiriam).

## Resposta

**Uma forma só, nos dois modos:** array no topo, na **mesma ordem** dos critérios, cada item sendo o
`rótulo` → lista de registros.

```json
[
  { "<rótulo_1>": [ { "...": "..." }, { "...": "..." } ] },
  { "<rótulo_2>": [] }
]
```

No **modo object** o array tem um único item, com o mesmo formato — o rótulo continua presente.

Garantias da resposta:
- **Ordem preservada** — o i-ésimo item da resposta corresponde ao i-ésimo critério da requisição.
- **Itens vazios aparecem como `[]`** — num `200`, um critério sem resultados ainda ocupa sua posição
  com lista vazia (o cliente deve tratar resultados mistos: alguns preenchidos, outros vazios).
- `200` — há **ao menos um** critério com resultado.
- `204` — **todos** os critérios retornaram vazio. Vale nos **dois** modos.
- **A lista pode não ser tudo.** Ela é limitada pelo teto de registros; quando há mais além do teto, o
  item traz `_truncated` (logo abaixo). **A ausência de `_truncated` é a garantia de que a lista está
  inteira.**

### ⚠️ Toda consulta tem teto — e omitir `_paging` não desliga o teto

Se você não mandar `_paging`, um limite padrão é aplicado: **500** registros no modo array, **1000** no
modo objeto. Uma consulta que casaria 4000 linhas devolve as primeiras 500, com `200`.

**Como distinguir "é isso tudo" de "foi cortado":** quando há mais além do teto, o item da resposta traz
`_truncated`, ao lado do rótulo:

```json
[
  {
    "<rótulo>": [ "..." ],
    "_truncated": true,
    "_maxRegisters": 500
  }
]
```

- A marca **só** aparece quando há mais — inclusive uma lista com exatamente o tamanho do teto vem
  **sem** ela, porque nesse caso não sobrou nada.
- Para continuar, repita a consulta avançando `_firstRegister` em `_maxRegisters`.
- **Ao percorrer as chaves do item, pule as que começam com `_`.** O rótulo da entidade é a que não
  começa; `_truncated` e `_maxRegisters` são controle de resposta, não dados.
- Para saber **quantos existem** antes de paginar, use `_count` — ele devolve
  `{"<rótulo>": {"totalRegisters": N}}` respeitando o mesmo filtro.

Detalhe dos controles em [query-controls.md](../query-controls.md).

> ⚠️ **`204` é decidido por haver REGISTRO, não por haver envelope.** Uma resposta que teria a forma
> `[{"<rótulo>": []}]` — envelope presente, lista vazia — é `204` **sem corpo**, não `200` com o envelope
> vazio. Só se algum critério trouxer ao menos um registro é que a resposta vem com corpo; nesse caso os
> demais aparecem com `[]`, como descrito acima.

> **Atributo do tipo `Json` volta como está gravado.** Objeto volta objeto; array de objetos volta array
> de objetos, com o aninhamento inteiro. Nada é achatado nem reembalado na leitura — é a mesma forma que
> o comando exige na escrita (ver [comando](../../persistence-crs/endpoints/comando.md)).

## Erros
`400` (JSON inválido), `401`/`403` (autorização), `510` (falha de execução da consulta).
Catálogo: [../erros.md](../erros.md).
