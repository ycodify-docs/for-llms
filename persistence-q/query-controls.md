# persistence-q · controles de consulta (DSL)

> Operadores de predicado e **controles** (`_paging`, `_sorting`, `_count`, `_cache`, `_connective`,
> associação) de um critério de consulta. Complementa [endpoints/consulta.md](endpoints/consulta.md)
> (modos array/object) e [README](README.md). Guia do serviço: [README.md](README.md).

## Contents
- Predicados e operadores
- Conectivo (AND/OR)
- Paginação · ordenação · contagem
- Cache
- Associações (popular relacionados)

---

## Predicados e operadores

Dentro de um critério (`{ "<rótulo>": { ...predicados } }`), cada predicado é um atributo com um valor
ou com um **operador**:

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

Exemplo:
```jsonc
{ "pedidos": { "total": { "gte": 1000 }, "cor": { "ilike": "%verm%" } } }
```

## Conectivo (AND/OR)

Múltiplos predicados num critério são combinados por `_connective` (padrão **`AND`**):

```jsonc
{ "pedidos": { "_connective": "OR", "status": "criada", "cor": "azul" } }
```

## Paginação · ordenação · contagem

Controles (chaves reservadas, prefixo `_`) dentro do critério:

```jsonc
{ "pedidos": {
    "status": "criada",
    "_paging":  { "_maxRegisters": 50, "_firstRegister": 0 },   // tamanho da página, deslocamento
    "_sorting": { "_orderBy": "criadaem", "_order": "DESC" },    // _order: ASC | DESC
    "_count":   false                                            // true → retorna a contagem (totalRegisters)
} }
```

- **`_paging`** — `_maxRegisters` = nº máximo de registros; `_firstRegister` = registro inicial (deslocamento).
- **`_sorting`** — `_orderBy` = campo; `_order` = `ASC`|`DESC`. Padrão: `_orderBy: id`, `_order: ASC`.
- **`_count`** — `true` devolve a **contagem** (`totalRegisters`) em vez das linhas.

## Cache

```jsonc
{ "pedidos": { "status": "criada", "_cache": { "_behavior": "use", "_ttl": <tempo> } } }
```

- **`_behavior`**: `use` (usa/popula o cache), `evict` (invalida), `ignore` (não usa).
- **`_ttl`**: tempo de vida da entrada em cache.

## Associações (popular relacionados)

Para trazer entidades **associadas** junto do resultado, use os controles de população, limitando a
profundidade por `_level`:

```jsonc
{ "pedidos": { "status": "criada", "_associations": [...], "_populating": true, "_level": 1 } }
```

- **`_associations`** / **`_populating`** — quais associações popular e se popula.
- **`_level`** — profundidade de população (evita expansão excessiva).
- **`_as`** — alias para o resultado.

> As chaves de controle têm prefixo `_`; nomes de atributo são `^[a-z]+$` (sem `_`), portanto não colidem.
