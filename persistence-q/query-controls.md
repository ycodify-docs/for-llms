# persistence-q · controles de consulta (DSL)

> Operadores de predicado e **controles** (`_paging`, `_sorting`, `_count`, `_cache`, `_connective`,
> associação) de um critério de consulta. Complementa [endpoints/consulta.md](endpoints/consulta.md)
> (modos array/object) e [README](README.md). Guia do serviço: [README.md](README.md).

## Contents
- Onde os controles ficam (leia primeiro)
- O rótulo é o nome da projeção
- Predicados e operadores
- Conectivo (AND/OR)
- Paginação · ordenação · contagem
- Cache
- Associações (popular relacionados)
- Quando o modelo é lido

---

## Onde os controles ficam (leia primeiro)

**Os controles são irmãos do rótulo, não filhos dele.** Ficam no mesmo nível do nome da projeção, ao
lado dele — nunca dentro do objeto de critério.

```jsonc
{
  "pedido": { "status": "criada" },                              // ← o critério
  "_connective": "AND",                                          // ← controles, IRMÃOS do rótulo
  "_paging":  { "_maxRegisters": 50, "_firstRegister": 0 },
  "_sorting": { "0": { "_orderBy": "criadaem", "_order": "DESC" } },
  "_count":   false
}
```

⚠️ **Controle escrito dentro do objeto do rótulo é ignorado em silêncio.** Não há erro: a consulta roda
com o valor padrão, e quem pediu página de 50 recebe a página padrão sem saber. A validação de nome de
atributo **não pega isso** — ela isenta toda chave iniciada por `_`, justamente para que controle novo
não quebre consulta que funciona.

Se a sua consulta ignora a paginação, a ordenação ou o conectivo que você mandou, **é quase sempre isto.**

## O rótulo é o nome da projeção

O rótulo tem de ser **exatamente** o nome da projeção no modelo. Não é um apelido livre nem o plural do
nome: `"pedido"` e `"pedidos"` são coisas diferentes, e o segundo não existe a menos que você o tenha
modelado assim.

Nome fora do modelo é recusado com **`400`**, e a mensagem lista os nomes declarados — use-a para achar a
grafia certa. A mesma regra vale para cada atributo dentro do critério.

## Predicados e operadores

Dentro de um critério (`{ "<rótulo>": { ...predicados } }`), cada predicado é um atributo com um valor
ou com um **operador**:

```jsonc
{ "<atributo>": "<valor>" }                 // igualdade simples
{ "<atributo>": { "<op>": "<valor>" } }     // operador explícito
```

Operadores (exaustivo): `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `ilike`, `in`, `distinct`.

| Operador | Significado |
|---|---|
| `eq` / `neq` | igual / diferente |
| `gt` / `gte` | maior / maior ou igual |
| `lt` / `lte` | menor / menor ou igual |
| `like` | padrão textual (case-sensitive); use curingas `%` (ex.: `"%ABC%"`) |
| `ilike` | como `like`, **case-insensitive** |
| `in` | pertence a um conjunto (lista de valores) |
| `distinct` | não filtra: pede os **valores distintos** daquele atributo. Substitui a seleção inteira — o resultado passa a ser a lista de valores únicos, não os registros |

Exemplo:
```jsonc
{ "pedido": { "total": { "gte": 1000 }, "cor": { "ilike": "%verm%" } } }
```

### Regras que valem para todos os predicados

- **`like` e `ilike` exigem o `%`.** Sem nenhum `%` no valor, a resposta é **`400`** — o serviço não
  adivinha se você queria início, fim ou contém.
- **`like` e `ilike` só valem para atributo textual.** Em qualquer outro tipo, **`400`**.
- **`%` num valor simples vira busca textual sem você pedir.** Na forma sem operador
  (`{"nome": "%ana%"}`), um valor que **começa** com `%` ou **termina** com `%` é tratado como
  `ilike`, não como igualdade. É o único lugar onde a plataforma troca o operador por conta própria —
  se você quer igualdade com um `%` literal, use `eq`.
- **`in` exige ao menos um valor.** Lista vazia é **`400`**.
- **Valor vazio ou só espaços não vira nada.** `{"cor": ""}` não filtra **e não traz a coluna** no
  resultado. Não é erro; é omissão silenciosa. Para "campo vazio", use `eq` com o valor que você quer
  comparar.

### Faixa (dois operadores no mesmo atributo)

Dois operadores no mesmo atributo formam uma **faixa**, e só nesta combinação:

```jsonc
{ "pedido": { "total": { "gte": 100, "lte": 500 } } }
```

- O primeiro tem de ser `gt` **ou** `gte` (limite inferior); o segundo, `lt` **ou** `lte` (limite
  superior). Qualquer outra dupla é **`400`**.
- Os dois lados são combinados por `AND`. Para combiná-los por `OR`, acrescente ao objeto do atributo a
  chave **`CONNECTIVE`** — em maiúsculas, sem o `_` — com valor `"OR"`. É a única chave de controle da
  DSL que não usa o prefixo `_`.
- Três ou mais operadores no mesmo atributo não são suportados.

## Conectivo (AND/OR)

`_connective` diz como os predicados se combinam. Padrão: **`AND`**.

```jsonc
{
  "pedido": { "status": "criada", "cor": "azul" },
  "_connective": "OR"
}
```

- **Só `AND` e `OR`, em maiúsculas.** Qualquer outro valor — inclusive `"and"` minúsculo — é **`400`**.
- **É global.** Vale para o critério inteiro e **propaga** para associações e componentes trazidos junto:
  não há como usar `AND` na entidade e `OR` numa associação dela na mesma consulta.

## Paginação · ordenação · contagem

Os três são **irmãos do rótulo** (veja a primeira seção).

### `_paging`

```jsonc
"_paging": { "_maxRegisters": 50, "_firstRegister": 0 }
```

- `_maxRegisters` = nº máximo de registros; `_firstRegister` = registro inicial (deslocamento).
- **Os dois campos são obrigatórios juntos.** Mandar só um é **`400`** — sem os dois a consulta sairia
  sem limite nenhum.
- **Omitir `_paging` não traz tudo:** um limite padrão é aplicado. Ele depende do modo da requisição —
  **500** no modo array, **1000** no modo objeto (veja [endpoints/consulta.md](endpoints/consulta.md)).
  Se você precisa de um número previsível, **mande `_paging`**.

#### Como saber que a resposta veio cortada

Quando existem mais registros do que o teto — o seu `_maxRegisters` ou o padrão —, o item da resposta
traz **`_truncated`**, ao lado do rótulo:

```jsonc
{
  "pedido": [ /* ... o teto de registros ... */ ],
  "_truncated": true,
  "_maxRegisters": 500        // o teto que foi aplicado
}
```

- **A marca só aparece quando há mais.** Ausência dela quer dizer **lista inteira** — inclusive quando a
  lista tem exatamente o tamanho do teto.
- Para continuar, repita a consulta avançando `_firstRegister` em `_maxRegisters`.
- **`_truncated` e `_maxRegisters` não são rótulo de entidade.** Se você percorre as chaves do item,
  pule as que começam com `_` — a entidade é a que não começa.

### `_sorting`

A forma é **indexada por posição**, com as chaves em string:

```jsonc
"_sorting": {
  "0": { "_orderBy": "criadaem", "_order": "DESC" },
  "1": { "_orderBy": "id",       "_order": "ASC"  }
}
```

- **Só as posições `"0"`, `"1"` e `"2"` são lidas** — no máximo três critérios de ordenação. Uma quarta
  posição é aceita e ignorada.
- **A forma plana (`{"_orderBy": …, "_order": …}` direto) é `400`.** Antes ela era aceita e a consulta
  saía **sem ordenação nenhuma**, sem aviso.
- `_orderBy` tem de ser um atributo **declarado no modelo** da entidade; fora dele, **`400`**.
- `_order` aceita apenas `ASC` e `DESC`. Ausente, vale `ASC`. Qualquer outro valor é **`400`**.
- Omitir `_sorting` ordena por `id`, `ASC`.
- ⚠️ **Entidade particionada ignora o seu `_orderBy`.** Se o modelo declara partição, a ordenação usa o
  campo da partição, seja qual for o atributo que você pediu. O `_order` que você mandou continua valendo.

### `_count`

```jsonc
"_count": true
```

`true` devolve a **contagem** em vez das linhas. O envelope é o mesmo; muda o que há dentro dele:

```jsonc
[ { "pedido": { "totalRegisters": 4217 } } ]
```

- O total vem sob **`totalRegisters`**, e não como um número solto.
- **A contagem respeita o mesmo filtro da consulta** — é o total que a consulta traria sem teto.
- Com `_count: true`, **`_paging` e `_sorting` não se aplicam** — contar não pagina nem ordena. Também
  não há `_truncated`: não há lista para cortar.
- Serve, portanto, para saber **quantos existem** antes de decidir como paginar.

## Cache

```jsonc
{
  "pedido": { "status": "criada" },
  "_cache": { "_behavior": "use", "_ttl": <tempo> }
}
```

- **`_behavior`**: `use` (usa/popula o cache), `evict` (invalida), `ignore` (não usa).
- **`_ttl`**: tempo de vida da entrada em cache. **Obrigatório com `_behavior: "use"`** — sem ele a
  chamada falha com erro de aplicação, não com `400`.
- **`_cache` sem `_behavior` é ignorado** por inteiro, sem erro. `_behavior` com valor desconhecido é
  **`400`**.

⚠️ **A chave da entrada inclui os controles que você mandou explicitamente.** Ela é formada a partir do
corpo da consulta **antes** de `_paging`, `_sorting`, `_connective` e `_count` serem retirados. Efeito
prático: a mesma consulta gravada **com** `_paging` explícito e relida **sem** ele produz chaves
diferentes — o resultado não é reaproveitado, e um `evict` mandado com controles diferentes dos da
gravação não alcança a entrada. **Para o cache funcionar, mande sempre o mesmo conjunto de controles.**

> ⚠️ **`_behavior: "use"` não devolve resultado correto em nenhum dos dois casos — não use este controle
> por ora.**
>
> - Na **falta** (a entrada não existe) a resposta vem **sem os registros e a consulta sequer é
>   executada**: o serviço não distingue "não achei" de "achei", e trata a ausência como acerto.
> - No **acerto** (a entrada existe) o cliente recebe o **invólucro** da entrada em vez do resultado —
>   formato diferente do da mesma consulta sem cache.
> - No **modo objeto**, além disso, a consulta ainda roda depois: a resposta volta com **dois** itens.
>
> O alcance é limitado porque o controle é **opt-in**: só afeta quem o declara. **Enquanto não houver
> correção, omita `_cache` ou use `_behavior: "ignore"`** — a consulta sem cache responde corretamente.
> O `evict` funciona: ele apaga a entrada, e passou a formar a chave do mesmo jeito que o `use` a formou
> (desde que mandado com os mesmos controles).

## Associações (popular relacionados)

**Não há controle declarativo de população.** A associação vem junto quando você a inclui no critério,
como um objeto aninhado sob o nome dela:

```jsonc
{ "pedido": { "status": "criada", "itens": [ { "quantidade": { "gte": 1 } } ] } }
```

- **Um item por consulta.** O critério de uma associação é atendido para **um** item; mandar dois ou
  mais é **`400`**. Antes o segundo era descartado em silêncio, e a resposta parecia completa.
- O `_connective` do topo **propaga** para a associação (veja a seção do conectivo).
- O vocabulário da associação é cobrado como o da entidade: nome fora do modelo é **`400`**.

> As chaves de controle têm prefixo `_`; nomes de atributo são `^[a-z]+$` (sem `_`), portanto não colidem.

## Quando o modelo é lido

A consulta só é atendida se o modelo do tenant estiver **ativo**. Esse estado é verificado **uma vez por
tenant em cada instância do serviço** — na primeira requisição daquele tenant —, não a cada consulta.

Consequências que importam na prática:

- **Voltar o modelo para edição não interrompe quem já está sendo atendido.** Uma instância que já
  atendeu aquele tenant continua respondendo às consultas dele com o modelo que carregou.
- **A mesma mudança pode "pegar" numa instância e não noutra**, conforme cada uma já tenha atendido o
  tenant ou não.
- **Mudança de modelo só vale com certeza para todos depois que as instâncias recarregam.** Se você
  precisa que a alteração valha imediatamente em todo lugar, trate isso como uma operação de
  implantação, não como um efeito da própria alteração.
