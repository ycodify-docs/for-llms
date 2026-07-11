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
    "forWriteModel.name": "wdb.client",   // valor FIXO definido pela plataforma (igual em todo agregado)
    "forReadModel.name":  "<dataschema>"  // = nome do dataschema da projeção (por padrão = nome do bc)
  },
  "tenantId": {
    "forWriteModel": "<tenant-id>",       // gerado na criação do dataschema (forger)
    "forReadModel":  "<tenant-id>"
  },
  "identity":    { "strategy": "uuid", "fields": [] },     // ver "Identidade e unicidade"
  "concurrency": { "version": "version", "strategy": "optimistic" },

  "command": { ... },
  "event":   { ... }
}
```

- **`schema.forWriteModel.name`** é um **valor fixo definido pela plataforma**, **igual em todo
  agregado** (o armazém de escrita é universal). Não é escolhido pelo autor do modelo. **Na publicação,
  o forger NORMALIZA este campo**: se o valor enviado não for o universal, ele é **convertido
  automaticamente** para o valor canônico (e, se ausente, é definido). Ou seja, **não confie** no valor
  que você enviar aqui — o serviço o impõe. (Nesta documentação o valor universal é representado como
  `wdb.client`.)
- **`schema.forReadModel.name`** é o **nome do dataschema** que hospeda a **projeção** do agregado —
  por padrão, igual ao nome do **bounded context** (ver
  [conceitos — do agregado à projeção](../../02-conceitos.md#do-agregado-à-projeção-derivação-e-implantação)).
- `concurrency.strategy: "optimistic"` → concorrência otimista (ver contrato do `status` no
  [README](../README.md#estados-transições-e-concorrência)).

> **⚠️ Runtime — o tenant vem do header, não deste campo:** em runtime a **fonte autoritativa** do
> tenant é o header **`X-Tenant-Id`** da requisição. Os campos `tenantId.forWriteModel`/`forReadModel`
> do JSON são **IGNORADOS** para a resolução de roteamento — `forReadModel` só é **injetado no payload
> do evento** publicado. Não modele assumindo que este campo do modelo dirige o roteamento por tenant.

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
  `command.…​.data.attribute`. Declara uma **constraint de unicidade composta**: a **combinação dos
  valores** desses atributos deve ser **única** entre os registros do agregado (não pode haver dois
  registros com a mesma combinação). `[]` = sem restrição de unicidade adicional além do `id`.
  - Ex.: `"fields": ["cnpj"]` → não há dois agregados com o mesmo `cnpj`.
  - Ex.: `"fields": ["razaosocial", "segmento"]` → o par (`razaosocial`, `segmento`) é único.

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
| `Date` | data | — |
| `Timestamp` | data e hora | — |
| `Json` | conteúdo semiestruturado (objeto/lista) | — |

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
- **`data.valueObject`** — dados **aninhados** (cada `<nomeVO>` é um grupo de campos tipados, mesma
  forma de um atributo). Na **projeção** (banco de leitura), cada value object vira **uma coluna `Json`**:
  - `valueObject.single.<nomeVO>` → **objeto JSON** (1:1);
  - `valueObject.multiple.<nomeVO>` → **array de objetos JSON** (1:N).
  Ou seja, os atributos de uma entity **não** vêm só de `data.attribute`: vêm de
  `data.attribute` (colunas escalares) **+** `valueObject.single`/`valueObject.multiple` (colunas JSON).
  Cada `<nomeVO>` pode ser um **grupo de campos** (`{ campo: {type,...} }`) **ou** um **único atributo
  tipado** direto (`{ type, ... }`) — neste caso, `multiple` é um array desse valor.
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
  gravação do evento. Convenção: particípio passado do verbo do comando + sufixo `em` (comando `criar` →
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
- `forWriteModel.name` é sempre o mesmo identificador universal; `forReadModel.name` é o esquema do contexto.
- Publicar o `.model.json` pelo forger ([model](../../forger/endpoints/model.md)) **não cria filas nem
  tabelas**: as projeções (tabelas) vêm das `entity` do forger; as filas, do deploy de processo.
