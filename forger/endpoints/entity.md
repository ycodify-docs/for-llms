# forger · endpoints · entity

> **entity** = definição de uma **projeção** materializada como **tabela no banco de leitura**. Criar/
> alterar uma entity passa pela **compilação** (léxica → sintática → semântica → representação
> intermediária → DDL) e aplica a DDL de forma **transacional** no banco de leitura. Guia: [../README.md](../README.md).
>
> A entity é a **projeção de um agregado**: suas colunas **derivam dos atributos do agregado** —
> escalares (`data.attribute`) **e** value objects (`valueObject.single` → coluna `Json` objeto;
> `valueObject.multiple` → coluna `Json` array) — e ela é criada **sob o dataschema** cujo nome = o do
> bounded context. Cadeia completa:
> [conceitos — do agregado à projeção](../../02-conceitos.md#do-agregado-à-projeção-derivação-e-implantação).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.
Caminho base: `/org/{org}/project/{project}/dataschema/{dataSchema}/entity`.

> **⚠️ Pré-condição de estado — o dataschema precisa estar em `MODELING`.** Criar/alterar/remover entity
> (e atributos/associações) só é permitido com o dataschema em `MODELING`. Em `RUNNING` o **forger
> rejeita** alterações de schema. Para editar um sistema já em operação: transite `RUNNING → MODELING`,
> edite, volte a `RUNNING`. Ver [dataschema — gate de status](dataschema.md#atualizar).

## Contents
- Regras de nome (entity, atributo, associação)
- Forma da definição
- Endpoints (exaustivo)
- Ciclo de vida (criar/atualizar)
- Declarar que a entity é projeção: `_conf.projectionOf` (teto e piso)
- Chave única: `attribute.unique` vs `_conf.uniqueKey`
- Semântica do `PUT`: ausente vs vazio
- Recorte de leitura por titular: `accessControl.scope`
- Regra de escopo único no `PUT`
- Saída de `analyze` / `validate`

## Regras de nome (entity, atributo, associação)

| Nome de | Caracteres | Padrão | Máx. |
|---|---|---|---|
| entity | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| atributo | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| associação | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| superentidade | só letras minúsculas | `^[a-z]+$` | **24** |

**Nomes reservados** (proibidos para entity/atributo/associação): `id`, `logversion`, `logrole`,
`loguser` — são atributos de controle injetados pela plataforma. Violar padrão/limite/reservado → `400`.

## Forma da definição

```json
{
  "name": "string",
  "attributes": [
    { "name": "string", "type": "String", "length": 60, "nullable": true,
      "unique": false, "comment": "string", "default": "string" }
  ],
  "associations": [
    { "name": "string", "targetEntity": "string", "nullable": true,
      "unique": false, "comment": "string" }
  ],
  "_conf": {
    "type": "entity",
    "comment": "string", "concurrencyControl": true,
    "uniqueKey": ["string"], "indexKey": ["string"],
    "accessControl": { "read": ["MASTER"], "write": ["MASTER"], "scope": {} },
    "projectionOf": ["string"],
    "superEntity": "string", "superEntityStrategy": "string"
  }
}
```

**Os dois campos `type` são de listas fechadas — e não são a mesma lista.**

| Onde | Valores aceitos | Obrigatório |
|---|---|---|
| `_conf.type` | `entity` · `abstract` · `component` · `enumeration` (minúsculas) | **sim** |
| `attributes[].type` | `String` · `Text` · `Integer` · `Long` · `Double` · `Float` · `Boolean` · `Date` · `Timestamp` · `Json` · `Jsonb` (**inicial maiúscula**) | **sim** |

> **⚠️ O tipo do atributo casa exatamente, com a caixa.** `"string"` em minúsculas é recusado, e nomes
> de tipo SQL não existem aqui: para decimal use **`Double`**, para texto longo **`Text`**. A recusa é
> `400` na compilação, antes de tocar o banco.
>
> **`length` é obrigatório para `String`** — os demais tipos o ignoram.

> **⚠️ `_conf` é OBRIGATÓRIO e semântico aqui.** A definição de **entity** exige `_conf` (configuração:
> chave única, índices, controle de concorrência, superentidade…). A convenção "chave `_`-prefixada =
> metadado removido" vale **apenas para o `.model.json`** (publicação de modelo), **não** para a
> definição de entity. **Não** remova `_conf` (nem outras chaves) deste payload — é outro contrato, em
> outro endpoint. Ver [model-format — chaves de metadado](../../persistence-crs/spec/model-format.md#chaves-de-metadado-_-prefixadas).

## Declarar que a entity é projeção — `_conf.projectionOf` (REGRA)

Uma entity **pode ou não** ser a projeção (read model) de um ou mais agregados (write model, do
`.model.json`). Ela declara isso em `_conf.projectionOf`:

```json
{ "name": "aula",
  "_conf": { "type": "entity", "projectionOf": ["aula", "matricula"] },
  "attributes": [
    { "name": "aggregateid", "type": "String", "length": 36, "nullable": false },
    { "name": "status", "type": "String", "length": 30, "nullable": false },
    { "name": "titulo", "type": "String", "length": 60, "nullable": true }
  ] }
```

| | |
|---|---|
| **ausente ou `[]`** | a entity **não é projeção** de agregado nenhum, e nada é conferido contra o modelo de escrita |
| **cada item** | é o **`type`** do agregado, **não** a chave do mapa `aggregate` do `.model.json` |
| **vários itens** | a entity é projeção da **união** dos agregados nomeados |

> ⚠️ **O item é o `type`, e a distinção não é cosmética.** No `.model.json` o agregado aparece sob uma
> chave composta (`"comercial.lead"`) e tem um campo `"type": "lead"`. O roteamento da projeção é feito
> pelo **`type` cru**. Então `"projectionOf": ["lead"]` casa e `"projectionOf": ["comercial.lead"]` é
> recusado — a mensagem de recusa lista os dois vocabulários.

### O teto e o piso: a projeção tem exatamente o que os agregados escrevem

Uma entity que se declara projeção é conferida nos **dois sentidos**, e um não implica o outro:

| | O que é recusado | Por que importa |
|---|---|---|
| **teto** | atributo que agregado nenhum dos nomeados escreve | coluna órfã, que fica sempre vazia |
| **piso** | atributo que os agregados escrevem e a entity **não** declara | **o grave**: sem a coluna, o motor recusa a gravação **inteira** e a linha nunca materializa |

> ⚠️ **Faltar é muito pior que sobrar, e não tem conserto.** O comando responde `200` porque a escrita
> funcionou; a projeção falha **depois**, em outro processo. Reemitir o comando falha, porque o agregado
> já mudou de estado — a linha fica divergente para sempre. Sobrar coluna só deixa espaço vazio.

O conjunto que um agregado escreve é:

- os **atributos de todos os `command`** (`command.<nome>.data.attribute`);
- o **nome de cada value object** (`data.valueObject.single` e `.multiple`) — o value object viaja
  **aninhado sob o próprio nome**, então ele é **uma** coluna, e os campos de dentro dele **não** são
  colunas da projeção;
- **um carimbo por evento** — o `whenAttribute` de cada `event`;
- **`aggregateid`** e **`status`**, que a plataforma escreve sempre.

Não contam contra o teto os atributos que a plataforma injeta (`id`, `logversion`, `logrole`,
`loguser`) nem as **associações** da entity. No **piso**, porém, uma **associação com o nome certo
satisfaz** — o motor aceita associação como chave do dado. A assimetria é deliberada: no teto a
associação é livre porque pode ter outra razão de existir; no piso ela resolve a gravação.

**A projeção não pode ser mais estreita que o agregado.** Não é escolha de desenho: uma chave que a
entity não tenha derruba a gravação inteira, não só aquele campo.

### Colunas obrigatórias de toda projeção — `aggregateid`, `status`, e os carimbos

- **`aggregateid`** — `String`, `length` **36**, `nullable: false`: o **UUID do agregado**; **identifica
  a linha** da projeção (ver [identificação na projeção](../../persistence-crs/spec/model-format.md#identificação-na-projeção-leitura)).
- **`status`** — `String`, `nullable: false`: o **estado atual** do agregado (campo de filtro típico em
  [persistence-q](../../persistence-q/README.md)).
- **cada `whenAttribute` de evento** — `Timestamp`, `nullable: true`: o carimbo do evento (ex.: o evento
  `criada` tem `whenAttribute` `criadaem`). Há **uma coluna por evento**. O valor é preenchido pela
  plataforma na escrita — por isso `whenAttribute` **não** é um `data.attribute` de comando no
  `.model.json`, mas a **coluna na projeção precisa existir**.

**Nenhum dos três é reservado nem auto-injetado.** O Forger injeta automaticamente **apenas** `id`,
`logversion`, `logrole` e `loguser`; `aggregateid`, `status` e os carimbos são **declarados pelo autor
da entity**. Faltando qualquer um, a projeção não materializa: o consumidor recusa a chave desconhecida
e a linha nunca aparece nas consultas de [persistence-q](../../persistence-q/README.md).

**O fechamento da edição recusa a entity a que falte qualquer uma delas** — é a conferência de piso
acima. `aggregateid` e `status` são recusados já na criação/atualização da entity; os carimbos só no
fechamento, porque só ali se sabe quais eventos o agregado tem.

### Onde a regra é imposta

| Momento | O que é recusado | Por quê ali |
|---|---|---|
| **criar/atualizar a entity** | `projectionOf` em `_conf.type` != `entity` · projeção sem `aggregateid` ou sem `status` · item vazio ou repetido | só depende da entity |
| **fechar a edição** (`PUT .../dataschema/{nome}` de `MODELING` para `RUNNING`) | agregado nomeado que não existe no modelo publicado · dois agregados com o mesmo `type` · **atributo acima do teto** · **qualquer coluna que os agregados escrevam e a entity não declare** — `aggregateid`, `status` e os carimbos incluídos | é o **único** instante em que a entity e o `.model.json` existem com certeza — os dois se publicam por caminhos diferentes e em qualquer ordem |

A recusa do fechamento acumula **todos** os achados numa resposta só, **nada é gravado**, e o dataschema
**continua em `MODELING`**. Conferir a cada publicação de artefato não é possível: recusaria a sequência
legítima em que o segundo artefato ainda não existe.

## Endpoints (exaustivo)

| Operação | Método · Path | Efeito | Sucesso |
|---|---|---|---|
| **Validar** | `POST .../entity/validate` | Compila a definição **sem persistir** (dry-run). | `200` |
| **Criar** | `POST .../entity` | Compila e aplica `CREATE TABLE` no banco de leitura. | `201` |
| **Ler** | `GET .../entity/{entity}` | Estado atual da definição. | `200` / `404` |
| **Listar** | `GET .../entities` | Todas as entities do esquema. | `200` / `404` |
| **Atualizar** | `PUT .../entity/{entity}` | Calcula e aplica o conjunto de mudanças (ChangeSet). | `200` / `409` |
| **+ Atributo** | `POST .../entity/{entity}/attribute` | Adiciona coluna (ALTER). | `200`/`201` |
| **+ Associação** | `POST .../entity/{entity}/association` | Adiciona associação. | `200`/`201` |
| **− Atributo** | `DELETE .../entity/{entity}/attribute/{attribute}` | Remove coluna. | `200` |
| **− Associação** | `DELETE .../entity/{entity}/association/{association}` | Remove associação. | `200` |
| **Remover entity** | `DELETE .../entity/{entity}` | `DROP` da tabela de projeção. | `200` |
| **Remover esquema (projeções)** | `DELETE /org/{org}/project/{project}/dataschema/{dataSchema}` | Remove o conteúdo de projeção do esquema no banco de leitura. | `200` |
| **Analisar** | `POST .../entity/{entity}/analyze` | Relatório de impacto de uma mudança (sem aplicar). | `200` |

## Ciclo de vida (criar/atualizar)

1. **Auth/validação** de caminho e corpo.
2. **Compilação** em estágios — o erro aponta **em qual estágio** a validação falhou:

   | Estágio | O que verifica | Erro típico |
   |---|---|---|
   | Léxica | tokens/estrutura básica do documento | caractere/símbolo inválido |
   | Sintática | forma (árvore): campos e aninhamento esperados | estrutura/campo ausente |
   | Semântica | tipos, ciclos, associações, referências cruzadas no contexto | tipo incompatível, referência inexistente, ciclo |
   | Representação intermediária | normaliza a definição validada | — |
   | DDL | gera as instruções a aplicar no banco de leitura | — |

3. **Aplicação transacional** da DDL no banco de leitura do tenant; qualquer erro → **rollback** total.
4. **Atualização do estado** da entity (para leituras subsequentes).
5. **Resposta**: `201` (criação) / `200` (alteração) **com o conjunto de mudanças aplicadas**
   (ver [Semântica do `PUT`](#semântica-do-put-ausente-vs-vazio)); falha de compilação/DDL → `400`/`500`.

## Chave única: `attribute.unique` vs `_conf.uniqueKey`

São dois conceitos **ortogonais**, e o serviço aceita os dois na mesma entity:

| Declaração | Significado |
|---|---|
| `attribute.unique: true` | unicidade no espaço de valores **daquele atributo** |
| `_conf.uniqueKey: ["a","b"]` | **chave composta** — exige 2+ atributos (com um só, é recusado) |

O uso conjunto é legítimo quando incidem sobre atributos **diferentes** — ex.: `email` único por si
**e** `(filial, codigo)` único como par. São duas chaves candidatas distintas.

> ⚠️ Quando o **mesmo** atributo é `unique` **e** aparece no `uniqueKey`, a composta fica
> **logicamente redundante**: se `a` já é único sozinho, o par `(a, b)` nunca rejeita nada que `a` já
> não rejeitasse. Não é erro — é aceito —, mas normalmente indica que a regra pretendida era só uma
> das duas. Custa um índice a mais em toda escrita e um passo a mais para remover a coluna (não se
> remove atributo que participa de `uniqueKey` sem antes tirá-lo de lá).
>
> O teste é uma pergunta: *duas linhas podem repetir o valor de `a` sozinho?* Se **sim**, `a` não pode
> ser `unique` — a unicidade dele só existe dentro do par.

A ordem declarada em `uniqueKey` não é preservada na constraint resultante; não conte com ela.

## Semântica do `PUT`: ausente vs vazio

O `PUT` é **atualização parcial**. Envie apenas o que quer mudar — e note que **omitir não é o mesmo
que esvaziar**:

| No payload | Efeito |
|---|---|
| propriedade **ausente** | **ignorada** — o valor atual é preservado |
| propriedade com **valor vazio** (`[]`, `""`) | **limpa** — o valor atual é removido |
| propriedade com valor | substituída |

```jsonc
// preserva uniqueKey, indexKey, comment… e aplica só o controle de acesso
{ "name": "ocorrencia", "_conf": { "type": "entity",
    "accessControl": { "read": ["MASTER","INSTRUTOR"], "write": ["MASTER","INSTRUTOR"] } } }

// remove a chave única composta (vazio explícito)
{ "name": "ocorrencia", "_conf": { "type": "entity", "uniqueKey": [] } }

// a entity deixa de ser projeção de agregado (vazio explícito)
{ "name": "aula", "_conf": { "type": "entity", "projectionOf": [] } }
```

> `projectionOf` segue a mesma regra: **ausente** preserva a declaração atual, `[]` a **remove**, e a
> entity volta a não ser conferida contra o modelo de escrita no fechamento da edição.

Vale igual para atributos e associações: declarar um atributo só com `nullable` altera **apenas**
`nullable` — `length`, `unique` e `comment` continuam como estavam.

Três detalhes que costumam morder:

- **`null` explícito conta como ausente.** Para limpar, use o vazio do tipo (`[]`, `""`), não `null`.
- **`accessControl` é mesclado**, não substituído em bloco: enviar só `read` **preserva** o `write`
  corrente, e vice-versa. Não existe entity sem controle de acesso — `MASTER` continua sendo o piso, e
  lista enviada vazia volta a `["MASTER"]`.
  *(Até **2026-09-09** o bloco era substituído por inteiro, e mandar só `read` zerava o `write` para
  `["MASTER"]` sem avisar. Se você integrou antes dessa data e manda as duas listas sempre "porque
  senão apaga", isso não é mais necessário — mas continua correto.)*
- **`_conf` é opcional no `PUT`.** Uma requisição que só toca atributos não precisa mencioná-lo.
  (Na **criação** ele continua obrigatório.) `name` é sempre obrigatório e deve casar com o caminho.

### Corpo da resposta

O `200` informa o que foi aplicado — uma alteração que remove uma constraint **diz isso na resposta**:

```jsonc
{
  "entityName": "ocorrencia",
  "valid": true,
  "summary": "…",
  "totalChanges": 1,
  "applied": [
    { "scope": "config",              // config | attribute | association
      "element": null,                // nome do atributo/associação, quando aplicável
      "property": "accessControl",
      "oldValue": { "read": ["MASTER"], "write": ["MASTER"] },
      "newValue": { "read": ["MASTER","INSTRUTOR"], "write": ["MASTER","INSTRUTOR"] },
      "description": "…",
      "ddl": "<operação aplicada>" }
  ],
  "skipped": []
}
```

Requisição sem efeito responde `200` com `totalChanges: 0` e `applied: []`.

## Recorte de leitura por titular: `accessControl.scope`

Declara que um **papel** lê apenas as linhas de que o usuário é titular. Sem ele, `accessControl` só
sabe dizer "este papel lê esta tabela" ou "não lê" — não existia forma de dizer "lê só o que é dele".

> **Disponibilidade:** no ar desde **2026-09-09**, em `yc-composer:amd64-260909` (`forger@421a8b8`), com
> a outra metade — o motor que honra o recorte — em **`yc-interpreter:amd64-260909b`**. **É opt-in:**
> enquanto nenhuma entity declarar `accessControl.scope`, nada muda para ninguém.
>
> **⚠️ O sufixo `b` importa, e não é detalhe de nomenclatura.** Houve duas imagens do interpreter no
> mesmo dia. Na primeira, `amd64-260909`, uma entity com recorte alcançada pela rota interna era
> **recusada com `403`** — declarar o `scope` ali quebraria os processors. A recusa saiu em
> `amd64-260909b`, e **só a partir dela o recorte é utilizável**. Se a sua superfície ainda está na
> imagem sem o `b`, **não declare o `scope`**.

```json
"_conf": {
  "accessControl": {
    "read": ["MASTER", "ADMIN", "OPERADOR"],
    "write": ["MASTER", "ADMIN"],
    "scope": {
      "read": { "OPERADOR": { "rows": { "by": "username" } } }
    }
  }
}
```

Mora **dentro** do `accessControl` porque é controle de acesso. Isso só é seguro porque o `PUT` do
`accessControl` mescla: se ele ainda substituísse o bloco, um `PUT` que trocasse papéis apagaria o
recorte da tabela inteira — e essa perda é **aberta**, a resposta continuaria `200` e com dados.

> **A regra, numa linha: `scope` só ESTREITA o que o `read` já permitiu.** Ele nunca concede acesso —
> papel que não está em `accessControl.read` não lê nada, e portanto nunca chega ao recorte. Por isso
> declarar `scope` para um papel fora do `read` é recusado (veja a tabela abaixo): não é o recorte que
> abre a porta, é o `read`; o recorte só decide **quanto** passa por ela.

**Semântica**

- papel **ausente** de `scope.read` lê sem recorte;
- `accessControl` **sem** `scope` é o comportamento de sempre;
- vários papéis casando, o **menos restritivo vence**;
- **`MASTER` nunca entra** no `scope` — ele é exigido nas duas listas e nunca é recortado;
- `by` nomeia um **atributo declarado** da entity, que guarda o titular da linha;
- para **remover** o recorte, envie `scope` vazio (`{}`) — isso não mexe nos papéis;
- o recorte vale **só na leitura**: veja abaixo o que acontece com `scope.write`.

**O que o `rows.by` precisa ser — e isto é decisão de modelagem, não de configuração**

O atributo tem de satisfazer as duas condições abaixo. Nenhuma delas é verificada na publicação: o
forger confere que o atributo **existe**, não o que ele **contém**.

1. **Conter o mesmo valor que identifica o usuário no token.** O recorte compara o conteúdo da coluna
   com a identidade de quem consulta; se a entity guarda `matricula` e o token identifica por
   `username`, a comparação nunca casa e o titular não vê as próprias linhas.
2. **Estar preenchido em toda linha que deva ser vista.** Linha com esse campo vazio não pertence a
   ninguém: ela **desaparece** para o próprio dono.

> **⚠️ As duas falham FECHADO, e é isso que as torna traiçoeiras:** a resposta é `200` com lista vazia,
> indistinguível de "não há dado". Ninguém recebe erro, nem quem consulta nem quem publicou a entity.
> Se um titular relata que "sumiu tudo", verifique estas duas antes de qualquer outra coisa.

Vale também para o caminho de escrita: se as linhas são criadas por outra pessoa que não o titular —
uma ficha de instrutor criada pelo administrador, por exemplo —, o atributo de titular precisa ser
**preenchido explicitamente com o titular**. É por isso que os metadados de auditoria não servem aqui:
eles registram quem escreveu.

**Recusado com `400` nesta versão**

| O que | Por quê |
|---|---|
| `MASTER` no `scope` | é o piso de toda entity e nunca é recortado |
| papel em `scope.read` fora de `accessControl.read` | papel que não lê não tem o que recortar — e um erro de digitação aqui significaria "sem recorte", falhando **aberto** |
| `rows.by` ausente | o recorte precisa saber por qual atributo cortar |
| `rows.by` com **caminho** (`assoc.atributo`) | reservado no formato, ainda não honrado pelo motor |
| `rows.by` nomeando `id`, `loguser`, `logrole`, `logversion`, `logdate` | são metadados da plataforma: registram **quem escreveu**, não **de quem é** a linha |
| `rows.by` que não é atributo declarado | — |
| a dimensão `attributes` | reservada no formato, ainda não honrada pelo motor |

**`scope.write` é caso à parte, e o comportamento depende do verbo** *(medido em
`yc-composer:amd64-260909`, 2026-09-09)*

O recorte só é honrado na **leitura**. Um `scope.write` não vazio:

- **na criação** da entity é **recusado com `400`**;
- **no `PUT`** é **aceito com `200` e o lado `write` é descartado em silêncio** — releia a entity e ele
  não está lá.

> **O resultado final é seguro; o que falta é o aviso.** A entity nunca fica se dizendo protegida na
> escrita sem estar — o `write` simplesmente não é gravado. Não há buraco de segurança aqui. O problema
> é que quem declarou acreditando ter protegido a escrita recebe `200` e **não é informado de nada**.
> **Confira relendo a entity depois do `PUT`:** se o `scope` voltar só com `read`, o seu `write` foi
> descartado.

O **efeito na consulta** — como o recorte se combina com filtros, `_connective` e `_count` — é do
`persistence-q`: veja a doc dele. Quem **declara** a chave é o forger; quem a **honra** é o motor.

## Regra de escopo único no `PUT`
Uma atualização (`PUT`) altera **um** escopo por requisição: **ou** `_conf`, **ou** atributos, **ou**
associações — nunca combinados na mesma chamada. Misturar escopos é erro comum de integração.

## Saída de `analyze` / `validate`
- **validate** compila sem persistir → `200` se a definição é válida; erro com o estágio, caso contrário.
- **analyze** devolve um **relatório de impacto** de uma mudança, com achados classificados por
  severidade (crítico / atenção / informativo) e recomendações. Achados **críticos** sinalizam que a
  mudança não deve ser aplicada como está.

> Use **validate** e **analyze** antes de aplicar mudanças destrutivas (remoção de coluna/associação).

## Coordenação
**CP-2:** as tabelas criadas aqui são as projeções que **persistence-q** consulta e que **persistence-crs**
(via es-n) atualiza. Crie as entities **antes** de publicar o **model**.
