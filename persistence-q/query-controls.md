# persistence-q · controles de consulta (DSL)

> Operadores de predicado e **controles** (`_paging`, `_sorting`, `_count`, `_connective`, `_cache`,
> associações) de um critério de consulta. Complementa [endpoints/consulta.md](endpoints/consulta.md)
> (modos array/object) e [README](README.md). Guia do serviço: [README.md](README.md).

## Contents
- Predicados e operadores
- Onde vão os controles (nível raiz × dentro do rótulo)
- Conectivo (AND/OR)
- Paginação · ordenação · contagem
- Cache
- Associações (popular relacionados)

---

## Predicados e operadores

Dentro do objeto do **rótulo** (`{ "<rótulo>": { ...predicados } }`), cada predicado é um atributo com um
valor ou com um **operador**:

```jsonc
{ "<atributo>": "<valor>" }                 // igualdade simples
{ "<atributo>": { "<op>": "<valor>" } }     // operador explícito
```

Operadores (exaustivo): `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `ilike`, `in`.

| Operador | Significado |
|---|---|
| `eq` / `neq` | igual / diferente |
| `gt` / `gte` | maior / maior ou igual |
| `lt` / `lte` | menor / menor ou igual |
| `like` | padrão textual (case-sensitive); use curingas `%` (ex.: `"%ABC%"`) |
| `ilike` | como `like`, **case-insensitive** |
| `in` | pertence a um conjunto (lista de valores) |

Exemplo (só predicados, dentro do rótulo):
```jsonc
{ "pedido": { "total": { "gte": 1000 }, "cor": { "ilike": "%verm%" } } }
```

## Onde vão os controles (nível raiz × dentro do rótulo)

⚠️ **REGRA DE NÍVEL — dois lugares distintos:**

- **No nível RAIZ do critério** (irmãos do rótulo, **fora** do objeto do rótulo): `_paging`, `_sorting`,
  `_count`, `_connective`, `_cache`.
- **Dentro do objeto do rótulo** (junto dos predicados): os **predicados** e os controles de associação
  (`_associations`, `_populating`, `_level`, `_as`).

Forma geral de um critério (modo object):
```jsonc
{
  "<rótulo>": { "<predicado>": "<valor>", "...": "..." },   // rótulo = nome da projeção; predicados aqui dentro
  "_connective": "AND",                                      // \_
  "_paging":  { "_maxRegisters": 50, "_firstRegister": 0 },  //  |— controles: nível RAIZ (irmãos do rótulo)
  "_sorting": { "0": { "_orderBy": "criadaem", "_order": "DESC" } },
  "_count":   false                                          // _/
}
```

> ⚠️ Controle **no lugar errado** não gera erro: se `_paging`/`_sorting`/`_count`/`_connective` forem postos
> **dentro** do objeto do rótulo, o serviço **não os enxerga** e aplica o **default** silenciosamente
> (página 1000/0, ordem por `id` ASC, `_count=false`, conectivo AND) — e ainda tenta interpretá-los como
> predicados. Mantenha-os no **nível raiz**.

## Conectivo (AND/OR)

`_connective` (nível **raiz**) define como **todos** os predicados do rótulo se combinam. Valores:
**`AND`** (padrão) ou **`OR`** — **maiúsculas**. É **global ao critério**: ou tudo AND, ou tudo OR.

Padrão (AND) — omitir `_connective`:
```jsonc
{ "pedido": { "status": "criada", "cor": "azul" } }
// registros com status = criada  E  cor = azul
```

OR (igualdades):
```jsonc
{ "pedido": { "status": "criada", "cor": "azul" }, "_connective": "OR" }
// registros com status = criada  OU  cor = azul
```

OR com operadores:
```jsonc
{ "pedido": { "total": { "gte": 1000 }, "cor": { "ilike": "%verm%" } }, "_connective": "OR" }
// registros com total ≥ 1000  OU  cor casando "%verm%" (case-insensitive)
```

OR + paginação + ordenação (critério completo):
```jsonc
{
  "pedido": { "status": "criada", "cor": "azul" },
  "_connective": "OR",
  "_paging":  { "_maxRegisters": 50, "_firstRegister": 0 },
  "_sorting": { "0": { "_orderBy": "criadaem", "_order": "DESC" } }
}
```

Notas:
- **Não é possível misturar** AND e OR no mesmo critério (o conectivo é único e global). Para lógica mista,
  emita critérios separados no **modo array** (cada um vira um resultado próprio; a combinação fica no cliente).
- Sub-condições **dentro de um mesmo atributo** (ex.: filtro composto sobre atributo do tipo objeto) são
  sempre combinadas por **AND**, independentemente do `_connective` do critério.

## Paginação · ordenação · contagem

Todos no nível **raiz**:

```jsonc
{
  "pedido": { "status": "criada" },
  "_paging":  { "_maxRegisters": 50, "_firstRegister": 0 },
  "_sorting": { "0": { "_orderBy": "criadaem", "_order": "DESC" } },
  "_count":   false
}
```

- **`_paging`** — `_maxRegisters` = nº máximo de registros; `_firstRegister` = registro inicial (deslocamento).
  Default: `_maxRegisters` 1000, `_firstRegister` 0.
- **`_sorting`** — objeto **indexado por posição** (`"0"`, depois `"1"`, `"2"` — até 3 chaves de ordenação,
  aplicadas nessa ordem). Cada uma: `_orderBy` = campo; `_order` = `ASC`|`DESC`.
  ⚠️ A forma **plana** (`{ "_orderBy": ..., "_order": ... }`, sem o índice `"0"`) é **ignorada em silêncio**
  (sem ordenação). Default (quando `_sorting` ausente): `_orderBy: id`, `_order: ASC`.
  Duas chaves:
  ```jsonc
  "_sorting": { "0": { "_orderBy": "criadaem", "_order": "DESC" }, "1": { "_orderBy": "id", "_order": "ASC" } }
  ```
- **`_count`** — `true` devolve a **contagem** (`totalRegisters`) em vez das linhas.

## Cache

`_cache` no nível **raiz**:
```jsonc
{ "pedido": { "status": "criada" }, "_cache": { "_behavior": "use", "_ttl": "<tempo>" } }
```

- **`_behavior`**: `use` (usa/popula o cache), `evict` (invalida), `ignore` (não usa).
- **`_ttl`**: tempo de vida da entrada em cache.

> **⚠️ Não use `_cache: use` por ora — o resultado vem errado.** Há um defeito conhecido, em conserto: na
> **primeira** chamada (nada em cache ainda) a consulta **não chega a ser executada** e a resposta vem com
> um objeto de controle no lugar dos registros; nas **seguintes**, o dado vem **embrulhado**, em formato
> diferente do que a mesma consulta devolve sem cache. Enquanto este aviso estiver aqui, prefira
> `_behavior: ignore` (ou simplesmente **omita** o `_cache`) — a consulta sem cache funciona normalmente.

## Associações (popular relacionados)

Para trazer entidades **associadas** junto do resultado, use os controles de população **dentro do objeto
do rótulo** (junto dos predicados), limitando a profundidade por `_level`:

```jsonc
{ "pedido": { "status": "criada", "_associations": [], "_populating": true, "_level": 1 } }
```

- **`_associations`** / **`_populating`** — quais associações popular e se popula.
- **`_level`** — profundidade de população (evita expansão excessiva).
- **`_as`** — alias para o resultado.

> As chaves de controle têm prefixo `_`; nomes de atributo são `^[a-z]+$` (sem `_`), portanto não colidem.
