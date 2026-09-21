# Formato do modelo de domínio (`.model.json`)

> Referência **formal** da estrutura do `.model.json` — o documento declarativo que descreve um
> agregado, seus comandos, eventos e transições. É o artefato central que um agente produz para fazer
> a plataforma operar um domínio. Fonte da verdade; em caso de dúvida, prevalece sobre a prosa.
> Guia: [../README.md](../README.md). Exemplos prontos: [../../examples/](../../examples/README.md).

> **Validação machine-readable:** [`model.schema.json`](model.schema.json) (JSON Schema draft-07) —
> os 6 exemplos em [../../examples/](../../examples/README.md) validam contra ele.

## Contents
- Estrutura de topo
- Nível do agregado
- Atributos e tipos
- Comandos
- Eventos e `domainBus`
- Estados e transições
- Convenções e regras

---

## Estrutura de topo

```jsonc
{
  "_comment": "texto livre (opcional) — metadado ignorado",   // ver "Chaves de metadado"
  "<org>.<project>": {              // chave = organização.projeto (um projeto é um bounded context)
    "aggregate": {
      "<bc>.<aggregateType>": { ... } // um ou mais agregados
    }
  }
}
```

### Chaves de metadado (`_`-prefixadas)

Qualquer chave que comece com **`_`**, **em qualquer nível** do `.model.json` (ex.: `_comment`, `_meta`,
`_schemaVersion`, `_source`), é **metadado não-semântico**: o forger as **remove recursivamente** (da
validação e do JSON publicado) e o grid de interpretação (persistence-crs/es-n) as ignora. Use
**`_comment`** para qualquer comentário livre no modelo — **nunca** um `comment` sem prefixo no topo
(seria confundido com a chave do bounded context e **rejeitado**).

> **Exceção — `comment` dentro de um atributo é semântico** (`data.attribute.<campo>.comment`): vira o
> **comentário da coluna** na projeção. Esse **não** leva `_` e **não** é removido. A regra `_` vale só
> para comentários/metadados **sem valor semântico** (topo do modelo, descrições).
>
> **Escopo:** esta convenção `_` é do **`.model.json`**. A definição de **entity** (forger) tem contrato
> próprio e usa **`_conf` obrigatório** (semântico) — lá `_` **não** é removido. Ver
> [forger/entity](../../forger/endpoints/entity.md).

## Nível do agregado

```jsonc
"<bc>.<aggregateType>": { // bc: nome do bounded context  '.' aggregateType: nome do tipo do agregado
  "type":           "<aggregateType>",  // nome do tipo do agregado (ex.: "pedido")
  "org":            { "name": "<org>" },
  "project":        { "name": "<project>" },
  "boundedContext": { "name": "<bc>", "comment": "<descrição opcional>" },

  "schema": {
    "forWriteModel.name": "wdb.client",   // carimbo do forger — não é lido em runtime
    "forReadModel.name":  "<dataschema>"  // carimbo do forger — não é lido em runtime
  },
  "tenantId": {
    "forWriteModel": "<tenant-id>",       // gerado na criação do dataschema (forger)
    "forReadModel":  "<tenant-id>"
  },
  "identity":    { "strategy": "uuid", "fields": [] },     // ver "Identidade e unicidade"
  "concurrency": { "version": "version", "strategy": "optimistic" },

  "roles": { ... },   // OPCIONAL — quem pode LER este agregado. Ver "Quem pode ler"

  "command": { ... },
  "event":   { ... }
}
```

- **`schema.forWriteModel.name`** e **`schema.forReadModel.name`** são **carimbados pelo forger na
  publicação** e **não dirigem o roteamento em runtime** — o grid de interpretação não lê nenhum dos
  dois. Declare-os (o schema os espera; o valor universal de escrita é representado aqui como
  `wdb.client`, e o de leitura, por convenção, é o nome do bounded context), mas **não modele contando
  com eles**: mudá-los não muda para onde nada vai. Quem decide de fato:

  | O que é decidido | Quem decide |
  |---|---|
  | banco do armazém de eventos | a **instância** que atende a rota do comando — não o modelo |
  | schema do armazém de eventos | **fixo**: `client` |
  | banco e schema da **projeção** | o **dataschema do tenant** (forger: `dataschema → database → dbconn`) |
  | schema do registro de consumo (`event_consumed`) | o **`boundedContext.name` do modelo** |

  Ver [conceitos — do agregado à projeção](../../02-conceitos.md#do-agregado-à-projeção-derivação-e-implantação).
- `concurrency.strategy: "optimistic"` → concorrência otimista (ver contrato do `status` no
  [README](../README.md#estados-transições-e-concorrência)).

> **⚠️ Runtime — o tenant vem do header, não deste campo:** em runtime a **fonte autoritativa** do
> tenant é o header **`X-Tenant-Id`** da requisição. Os campos `tenantId.forWriteModel`/`forReadModel`
> do JSON são **IGNORADOS** para a resolução de roteamento — `forReadModel` só é **injetado no payload
> do evento** publicado. Não modele assumindo que este campo do modelo dirige o roteamento por tenant.

### A chave do agregado — `<bc>.<type>`, e ela vale em três lugares

A chave sob `aggregate` **não é rótulo livre**: tem de ser exatamente
**`<boundedContext.name>.<type>`**, com os dois valores declarados dentro do próprio agregado. A mesma
string aparece em três lugares, e os três têm de concordar:

| Onde | Forma |
|---|---|
| chave do agregado no `.model.json` | `"vendas.pedido": { "type": "pedido", "boundedContext": { "name": "vendas" } }` |
| chave que o cliente envia no **comando** | `{ "vendas.pedido": { "<comando>": … } }` |
| schema PG do registro de consumo | `vendas.event_consumed` |

> **⚠️ Divergência aqui não dá erro — dá silêncio.** Se a chave não for exatamente
> `<boundedContext.name>.<type>`, o evento do agregado **nunca é processado**: não há resposta de erro,
> não há entrada de log e o comando parece ter sido aceito. O sintoma é **projeção que nunca aparece**,
> indistinguível de "nada aconteceu". Ao ver isso, **confira a chave antes de investigar qualquer outra
> coisa**.

> **⚠️ `boundedContext.name` deve ser o nome do dataschema do tenant.** A linha da projeção é gravada no
> dataschema **do tenant**, enquanto o registro de consumo é gravado no schema que leva o nome do
> **bounded context**. Quando os dois nomes não coincidem, as duas gravações vão para schemas
> diferentes — e o segundo pode nem existir no banco do cliente.

### Quem pode ler o agregado (`roles`)

```jsonc
"roles": {
  "read":   ["ASSOCIADO", "ADMINISTRADOR"],   // podem ler o agregado
  "author": ["ADMINISTRADOR"],                // veem QUEM fez cada evento no /history
  "scope":  {                                  // recorte por proprietário, opcional
    "read": { "ASSOCIADO": { "rows": { "by": "username" } } }
  }
}
```

**Sem esta chave, qualquer usuário do tenant lê o agregado.** É a declaração que liga o recorte.

| Chave | O que declara |
|---|---|
| `read` | os papéis que alcançam o agregado. Quem não tem nenhum deles não o enxerga |
| `scope.read.<PAPEL>.rows.by` | o atributo cujo valor precisa ser o **`username` do token** de quem pede — o proprietário |
| `author` | os papéis que veem, no `/history`, quem executou cada evento |

Os códigos de resposta estão em
[leitura de agregado — quem pode ler](../endpoints/agregado-leitura.md#quem-pode-ler).

> **A mesma palavra vale duas coisas, e o nível diz qual.** `roles` **no agregado** é leitura; `roles`
> **dentro de um comando** é escrita — quem pode executá-lo — e é um array. Ler pode caber a um papel que
> nunca escreve.

**`by` nomeia um atributo declarado** em `data.attribute` de algum comando: é de lá que o dado nasce. A
convenção para o atributo de proprietário é **`username`**.

**Papéis acumulados: o menos restritivo vence.** Basta um papel do solicitante estar em `read` sem recorte
declarado em `scope` para não haver recorte — quem acumula ADMINISTRADOR e um papel recortado não perde o
que o papel maior lhe dá.

> **⚠️ `loguser` não serve como `by`, e a publicação recusa.** Ele registra **quem executou o comando**,
> não **de quem o agregado é**: um cadastro feito pelo administrador leva o login dele, e o recorte
> devolveria nada ao dono do dado.

> **⚠️ Atributo de titular vazio = agregado invisível para o próprio dono.** Se o modelo recorta por
> `username` e o agregado tem esse campo em branco, ele não casa com ninguém — e a resposta é a de um id
> inexistente. Preencher o titular é parte da modelagem, em especial quando quem cadastra é outra pessoa.

### `loguser`: quem executou cada comando

Todo comando executado com credencial carimba **`loguser`** no dado do agregado, com o `username` de quem
o executou. É campo **da plataforma**: não se declara no modelo e não se envia no comando — declará-lo é
recusado, como os demais metadados.

Em comando disparado por **coordenação**, o valor é o de quem assinou o **primeiro** comando da cadeia: a
identidade atravessa a saga.

> `loguser` responde *"quem escreveu"*; `username`, quando o modelo o declara, responde *"de quem é"*. Os
> dois coincidem quase sempre, e divergem no caso que importa — o cadastro feito por terceiro.

### Identidade e unicidade (`identity`)

```jsonc
"identity": {
  "strategy": "uuid",                 // tipo do atributo `id` do agregado
  "fields":   ["<atributo>", "..."]   // chave de unicidade composta (nomes de atributos)
}
```

- **`strategy`** — define o **tipo do `id`** do agregado. `"uuid"` ⇒ o `id` é um UUID **gerado pela
  plataforma**; **não** o envie no comando de criação.
- **`fields`** — **vetor de strings**, em que **cada elemento é o nome de um atributo** declarado sob
  `command.…​.data.attribute`. Declara **qual combinação de valores identifica o agregado no negócio** —
  a resposta a *"o que faz dois destes serem o mesmo?"*. `[]` = nenhuma; o agregado é identificado só
  pelo `id`.
  - Ex.: `"fields": ["cnpj"]` → dois agregados com o mesmo `cnpj` são o mesmo.
  - Ex.: `"fields": ["razaosocial", "segmento"]` → o par identifica.

**A combinação é imposta na criação:** um segundo agregado com a mesma combinação de valores é
**recusado** — não nasce. Modelo com `fields: []` não tem essa restrição.

> **⚠️ Campo declarado que não vem no comando desliga a unicidade daquele agregado.** Se **qualquer**
> atributo listado em `fields` estiver ausente ou nulo no dado do comando de criação, o agregado é
> criado **sem chave de unicidade** — e uma repetição futura com os mesmos valores passa a ser aceita.
> Não há recusa e não há aviso na resposta. Portanto: **todo atributo listado em `fields` tem de ser
> obrigatório no comando de criação**, e é assim que se modela.

**O que acontece na colisão.** A criação do segundo é recusada e o agregado não nasce. Quando a
criação parte de uma **coordenação**, a plataforma reconhece a recusa como *"o alvo já existe"* e
considera a coordenação **concluída** — não é erro, e não vai para fila de descarte.

#### Como escolher os `fields` — e o erro que a escolha ingênua produz

A pergunta a responder não é *"quais campos são obrigatórios?"*, é **"o que faz dois destes serem o
mesmo?"**. E há uma armadilha frequente: **recriar depois de encerrar costuma ser legítimo.**

Exemplo. Uma matrícula com `fields: ["aulaid", "alunoid"]`:

| Quando | O que acontece | Com essa chave |
|---|---|---|
| março | o aluno se matricula na turma | ok |
| junho | o aluno sai; a matrícula é encerrada | ok |
| agosto | o aluno **volta para a mesma turma** | **seria recusado** — mesma combinação |

Voltar em agosto é uma matrícula **nova e legítima**. Se a chave não a distingue da de março, a
imposição — quando existir — passaria a recusar operação correta, que é pior do que aceitar a repetida.

**A correção é de modelagem, não da plataforma:** a chave precisa conter o que separa uma da outra —
o período letivo, a data de início, a temporada. Se duas coisas podem coexistir legitimamente com os
mesmos valores, **os `fields` estão incompletos**.

> A plataforma garante *"esta combinação não se repete"*. **Quais campos formam a combinação é decisão
> de quem modela** — e é onde o domínio entra.

### Identificação na projeção (leitura)

Na **projeção** (banco de leitura), a linha de cada agregado é **identificada por `aggregateid`** — o
`id` (UUID) do agregado. Consultas que buscam um agregado específico filtram por esse campo; o fluxo de
projeção localiza/atualiza a linha por ele. O atributo **`status`** carrega o **estado atual** do
agregado (ver [regra do `status`](../README.md#estados-transições-e-concorrência)). Em consultas
([persistence-q](../../persistence-q/README.md)), `aggregateid` e `status` são campos típicos de filtro.

> **`id` tem tipo por contexto (contrato de identidade):**
> - **Projeção (leitura):** a linha tem **dois** identificadores — `id` (**Long**, PK da linha no
>   read-model, auto-injetado pelo Forger) e `aggregateid` (**UUID** de 36 chars, o id do agregado).
>   Para localizar a projeção de **um** agregado específico, filtra-se por `aggregateid`.
> - **Agregado (persistence-crs):** nos endpoints `/a/{bc}/{type}/{id}` (comando/transição, leitura de
>   estado, `/history`) o `{id}` é o **UUID** do agregado.
> - **Identidade:** `projecao.aggregateid == aggregate.id` (o `aggregateid` da projeção é o mesmo UUID).
> - **Falha:** enviar o `id` (Long) da projeção onde a persistence-crs espera o UUID → `510`
>   "Invalid UUID string". Ex.: `GET /a/vendas/pedido/1` falha; `GET /a/vendas/pedido/<uuid>` funciona.

## Atributos e tipos

Atributos aparecem em `command.<cmd>.data.attribute.<campo>`:

```jsonc
"<campo>": {
  "type":     "String",      // ver vocabulário abaixo
  "length":   300,            // tamanho máximo — OBRIGATÓRIO p/ type:"String" (N/A p/ os demais tipos)
  "nullable": false,          // se aceita nulo
  "comment":  "<descrição opcional>"
}
```

**Vocabulário de tipos** (exaustivo — não inventar outros):

| Tipo | Uso | `length`? |
|---|---|---|
| `String` | texto curto/médio | **sim — OBRIGATÓRIO** (o Forger exige; sem `length` o publish/entity falha) |
| `Text` | texto longo | — |
| `Integer` | inteiro | — |
| `Long` | inteiro grande (também usado p/ valores monetários em menor unidade, ex.: centavos) | — |
| `Boolean` | verdadeiro/falso | — |
| `Date` | data de calendário, **sem fuso** — gravada e devolvida como `2026-09-13` | — |
| `Timestamp` | data e hora, **em UTC, sem fuso** — gravada e devolvida como `2026-09-12T20:19:09` | — |
| `Json` | conteúdo semiestruturado (objeto/lista) | — |

### Datas: `Timestamp` e `Date`

**Por que isto importa:** a mesma data sai por dois caminhos — o estado do agregado e a projeção. Se cada
um a devolvesse numa grafia, o front teria de reconciliar as duas. Por isso **toda data é normalizada ao
entrar no comando**, e as duas leituras devolvem **a mesma string**.

**A convenção:** `Timestamp` é gravado e devolvido **em UTC, sem fuso**; a conversão para o horário de quem
lê é do front. `Date` é dia do calendário e **não tem fuso**.

| Tipo | Você pode enviar | Fica gravado e volta como |
|---|---|---|
| `Timestamp` | `"2026-09-15T06:00:00-03:00"` — com fuso | `"2026-09-15T09:00:00"` — convertido para UTC |
| | `"2026-09-15T09:00:00Z"` · `"…+00:00"` | `"2026-09-15T09:00:00"` |
| | `"2026-09-15 09:00:00"` · `"2026-09-15T09:00:00.665"` — **sem fuso** | `"2026-09-15T09:00:00"` — **lido como UTC**, fração descartada |
| | `"2026-09-15"` — só data | `"2026-09-15T00:00:00"` — início do dia em UTC |
| | `1789462800000` — número | `"2026-09-15T09:00:00"` — epoch em milissegundos |
| `Date` | `"1990-04-02"` | `"1990-04-02"` |
| | `"1990-04-02T23:00:00-03:00"` — com hora | `"1990-04-02"` — **o dia como escrito**, sem conversão |

> ⚠️ **O erro mais comum: hora local sem fuso.** `"2026-09-15 06:00:00"` é **06h UTC** — 03h em Brasília.
> Se a hora veio de um formulário, mande `-03:00` junto.

> **Por que `Date` não converte:** um aniversário registrado às 23h de Brasília seria dia seguinte em UTC.
> Quem manda uma data manda um dia do calendário, não um instante — converter mudaria o dia.

**Onde a normalização acontece:** em todo atributo declarado `Timestamp` ou `Date`, **solto ou como campo de
valueObject em grupo**, e **também na data que a regra de negócio devolver** — ela é aplicada de novo depois
da regra.

**Onde NÃO acontece — e a data segue exatamente como você mandou:**
- dentro de valueObject de **tipo direto** (`{"type": "Json"}`): o modelo não diz que há data lá dentro;
- em campo `String`, mesmo que o texto seja uma data ou uma hora (`"06:00"`);
- em dado gravado **antes** de `yc-interpreter:amd64-260913c` — a imagem em que isto passou a valer em teste;
  **produção ainda não tem**, e lá nenhuma data é normalizada.

**Recusas:** forma irreconhecível — `"13/09/2026"`, `"ontem"`, epoch num `Date` — é **`400`** que nomeia o
campo (`no campo 'notafiscal.datacompra': …`). `"13/09/2026"` é recusado de propósito: dia e mês trocam de
lugar conforme o país. Valor **vazio ou nulo** passa como veio.

Para filtrar por data numa consulta:
[persistence-q § Filtrar por data e hora](../../persistence-q/query-controls.md#filtrar-por-data-e-hora-date-timestamp).

> **Não há tipo decimal/float.** Para valores fracionários (ex.: dinheiro), use `Long` (menor unidade)
> ou `String`. `Json` cobre estruturas aninhadas livres.

## Comandos

```jsonc
"command": {
  "<nomeDoComando>": {
    "data": {
      "attribute": {
        "<campo>": { "type": "...", "length": 0, "nullable": true, "comment": "..." }
      },
      "valueObject": {
        "single":   { "<nomeVO>": { "<campo>": { "type": "...", "length": 0, "nullable": true } } },
        "multiple": { "<nomeVO>": { "<campo>": { "type": "...", "length": 0, "nullable": true } } }
      }
    },
    "fromState": ["<estado de origem>", "..."],  // [] no comando de criação
    "endState":  "<estado de destino>",
    "roles":     ["<papel>", "..."],             // papéis autorizados a executar
    "coordination": { },                          // {} se não houver coordenação síncrona
    "br": { "route": "<rota do processador>" }    // opcional: regra de negócio
  }
}
```

- **Criação:** `fromState: []` (não existe estado anterior).
- **`status` em comandos de transição (REGRA — obrigatório, exceto criação):** **todo** comando de um
  agregado, **exceto o de criação** (`fromState: []`), **DEVE declarar** o atributo **`status`**
  (`type: "String"`) em `data.attribute` — o **estado atual** do agregado, base da **concorrência
  otimista** da transição. O comando de **criação NÃO** declara `status` (o agregado ainda não existe).
  Vale para **todo agregado, em todo bounded context**.
- **`data.attribute`** — atributos **escalares** do agregado (uma coluna por atributo na projeção).
  **Não** inclua aqui o `whenAttribute` do evento (o carimbo, ex.: `criadaem`): ele é **auto-valorizado**
  e **não** é atributo de dados do comando — mas **é** coluna obrigatória da projeção (declarada na
  entity; ver [Eventos e `domainBus`](#eventos-e-domainbus) e [forger — entity](../../forger/endpoints/entity.md)).
- **`data.valueObject`** — dados **aninhados**. `valueObject.single.<nomeVO>` é 1:1 e
  `valueObject.multiple.<nomeVO>` é 1:N. Cada um vira **uma coluna** na projeção (banco de leitura) —
  ou seja, os atributos de uma entity **não** vêm só de `data.attribute`: vêm de `data.attribute`
  (colunas escalares) **+** `valueObject.single`/`valueObject.multiple`.

  Cada `<nomeVO>` tem **duas formas de declaração**, e a forma escolhida determina a **forma do valor
  no comando**. O discriminador é a chave `type` na raiz da declaração (nome de campo nunca é `type`:
  nomes casam `^[a-z]+$` e são do domínio):

  | Declaração | Forma | Valor em `single` | Valor em `multiple` | Coluna |
  |---|---|---|---|---|
  | `{ "<campo>": { "type": … } }` | **grupo de campos** | objeto | array de objetos | `Json` |
  | `{ "type": …, "length": … }` | **atributo tipado direto** | objeto | array de objetos | `multiple`: `Json` · `single`: o tipo do escalar |

  Repare que a coluna do valor **não muda entre as duas declarações**: seja qual for a forma de declarar,
  o valor é **objeto** em `single` e **array de objetos** em `multiple`.

  ```jsonc
  // grupo de campos                          // atributo tipado direto
  "multiple": {                               "multiple": {
    "itens": {                                  "diassemana": {
      "nome": { "type": "String" },               "type": "String", "length": 10, "nullable": false
      "qtd":  { "type": "Integer" }             }
    }                                         }
  }
  // comando:                                 // comando:
  // "itens": [ { "nome": "a", "qtd": 1 } ]   // "diassemana": [ { "dia": "terca" } ]
  ```

  > ### Duas formas, e só essas duas
  >
  > | | Valor |
  > |---|---|
  > | `valueObject.single` | **um objeto** |
  > | `valueObject.multiple` | **um array de objetos** |
  >
  > **Escalar nunca** — nem solto, nem dentro de array. `"diassemana": "terca"` e
  > `"diassemana": ["terca"]` são recusados com **`400`**.
  >
  > **O que existe dentro do objeto é seu.** Um campo ou dez, valores simples ou objetos aninhados: a
  > plataforma cobra a **forma**, não o conteúdo. Estes são todos válidos:
  >
  > ```jsonc
  > "endereco":   { "rua": "das Flores", "numero": 42 }                       // single, dois campos
  > "endereco":   { "rua": "das Flores", "cidade": { "nome": "Natal" } }      // single, aninhado
  > "diassemana": [ { "dia": "terca" }, { "dia": "quinta" } ]                 // multiple
  > "horarios":   [ { "periodo": { "de": "08:00", "ate": "12:00" } } ]        // multiple, aninhado
  > ```
  >
  > O aninhamento **chega intacto** ao agregado e volta intacto na consulta — nada é achatado nem podado
  > no caminho.
  >
  > ⚠️ **Um único item fora da forma reprova o comando inteiro.** Num `multiple`, basta um escalar no
  > meio de objetos (`[{"dia":"terca"}, "quinta"]`) para o comando ser recusado. Não há aproveitamento
  > parcial: ou o payload está na forma, ou é erro.

  **Comportamento do persistence-crs ao receber o comando:**
  - **Grupo de campos:** campos do item que **não** estejam declarados no modelo são **descartados**
    silenciosamente — o comando segue com o que foi declarado.
  - **Tipo direto:** o valor é **objeto** (ou array de objetos), e chega assim ou é `400` —
    `"diassemana": [{"dia": "terca"}]` grava como veio; `"diassemana": ["terca"]` é recusado. O conteúdo
    do objeto não é filtrado: vai como você mandou. A leitura devolve o que está gravado, sem reembalar
    — objeto volta objeto, array volta array.
  - **Forma incompatível → `400`** com a forma esperada (não `510`): `single` que recebeu escalar ou
    array; `multiple` que não recebeu array; e, em qualquer das duas declarações, item de `multiple` que
    não seja objeto.
  - **Coluna da projeção:** o valor é gravado como chega. Array e objeto exigem coluna `Json`; um
    `single` de **tipo direto** grava um **escalar**, então declare a coluna com o tipo do escalar
    (`String`, `Integer`, …) — numa coluna `Json`, um escalar de texto não é JSON válido.
- `roles`: lista de papéis que podem executar o comando.
- `br.route`: se presente, o persistence-crs chama o [br-service](../../br-service/README.md) antes de
  gravar o evento (CP-6). A rota segue a **forma canônica totalmente qualificada**
  `<org>/<project>/<bc>/<aggregate>/<comando>` (evita colisão entre organizações) — ver
  [br — forma canônica da rota](../../br-service/README.md#forma-canônica-da-rota-obrigatória).

## Eventos e `domainBus`

```jsonc
"event": {
  "<nomeDoEvento>": {
    "type": "<nome-do-evento>",          // identificador do tipo do evento
    "whenAttribute": "<campoTimestamp>", // atributo de carimbo de tempo (auto-valorizado)
    "payloadInherit": "command-data",    // herda os dados do comando
    "domainBus": {
      "orderingMode": "per-aggregate",    // none | strict | per-aggregate (padrão)
      "triggerProjection":   [ /* projeção cross-contexto, ver abaixo */ ],
      "triggerCoordination": [ /* saga, ver abaixo */ ]
    }
  }
}
```

- **`whenAttribute`** — nome de um atributo de timestamp **valorizado automaticamente** pela plataforma na
  gravação do evento, **em UTC**, na mesma forma dos demais `Timestamp` (`2026-09-12T20:19:09`). Convenção: particípio passado do verbo do comando + sufixo `em` (comando `criar` →
  evento `criada` → `whenAttribute: "criadaem"`). **Não** envie esse campo no comando **e NÃO o declare**
  como `command.<cmd>.data.attribute` — é auto-valorizado, nunca um atributo de dados do comando.
  - **Coluna obrigatória na projeção (REGRA):** ainda assim, **cada** `whenAttribute` **DEVE existir como
    coluna** (`type: "Timestamp"`, `nullable`) na **entity** do agregado — o autor/deployer da entity a
    **declara explicitamente** (como `aggregateid`/`status`), pois **não** deriva de `data.attribute` nem
    é auto-injetada. Se faltar, a projeção do agregado **não materializa** (leitura vazia). Ver
    [forger — entity / colunas obrigatórias](../../forger/endpoints/entity.md).
- **`domainBus`** é onde mora **todo** o roteamento de despacho (não no topo do evento):
  - `orderingMode` — ordenação da projeção (ver [es-n](../../es-n/README.md)).
  - `triggerProjection[]` e `triggerCoordination[]` — itens com a forma:
    ```jsonc
    { "name": "<alvo>", "targetTenantId": "<tenant-id destino>", "br": { "route": "<rota>" } }
    ```
  - Listas **vazias** = sem despacho extra (só a projeção do próprio contexto).

### O que o processor de coordenação devolve

O processor apontado por `triggerCoordination[].br.route` devolve **`processedData.targetCommand`**, com
as quatro chaves `boundedContext`, `aggregateType`, `commandName` e `data`:

```jsonc
{ "processedData": { "targetCommand": {
    "boundedContext": "estoque", "aggregateType": "itemestoque",
    "commandName":    "registrarentrada", "data": { "quantidade": 3 } } } }
```

> **⚠️ Resposta `200` com outra forma não dispara nada.** O comando-alvo não é submetido, não há erro,
> não há retentativa e não há sinal do lado de fora: o alvo fica intacto, como se a coordenação não
> existisse. Ao investigar *"a coordenação não faz nada"*, **confira a forma do retorno primeiro**.

**Um alvo por coordenação.** `targetCommand` é um objeto, não uma lista: quem precisa atingir N agregados
emite os N comandos dentro do próprio processor. O `coordination` declarado **dentro de um comando** é
outro mecanismo — síncrono, transacional — e esse é uma lista ordenada.

**O autor atravessa a saga.** O comando-alvo é gravado com a identidade de quem assinou o **primeiro**
comando da cadeia, e o papel dele não é reavaliado no alvo: comando disparado por comando já executado é
considerado autorizado. É essa identidade que aparece no
[`loguser`](#loguser-quem-executou-cada-comando) e no `/history` do agregado derivado.

> Declarar `triggerProjection`/`triggerCoordination` **não cria filas** — apenas roteia o despacho pelos
> canais universais do es-n. Ver [arquitetura — de onde vêm as filas](../../01-arquitetura.md#origem-das-filas).

## Estados e transições

- **Não há bloco `state` explícito.** Os estados são **strings** referenciadas em `fromState`/`endState`
  dos comandos.
- O conjunto de estados de um agregado é o conjunto de todos os `endState` (mais o "inexistente" antes
  da criação). Cada `fromState` de um comando deve ser `endState` de algum outro comando — máquina de
  estados conexa.

## Convenções e regras

- Nomes de comando/evento/atributo/valueObject seguem **`^[a-z]+$`** — apenas letras minúsculas do
  alfabeto português, **sem** `_`, dígito, acento ou separador; para **atributo**, **máx. 24** chars. O
  `whenAttribute` segue a convenção particípio+`em` (e também casa `^[a-z]+$`).
- O `id` do agregado e os campos de controle são gerenciados pela plataforma — **não** declarar/enviar.
- `schema.forWriteModel.name` e `schema.forReadModel.name` são carimbos do forger e **não dirigem nada**
  em runtime — ver [nível do agregado](#nível-do-agregado).
- Publicar o `.model.json` pelo forger ([model](../../forger/endpoints/model.md)) **não cria filas nem
  tabelas**: as projeções (tabelas) vêm das `entity` do forger; as filas, do deploy de processo.
