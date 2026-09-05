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
- Chave única: `attribute.unique` vs `_conf.uniqueKey`
- Semântica do `PUT`: ausente vs vazio
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
    { "name": "string", "type": "string", "length": 0, "nullable": true,
      "unique": false, "comment": "string", "default": "string" }
  ],
  "associations": [
    { "name": "string", "targetEntity": "string", "nullable": true,
      "unique": false, "comment": "string" }
  ],
  "_conf": {
    "comment": "string", "concurrencyControl": true,
    "uniqueKey": ["string"], "indexKey": ["string"],
    "accessControl": {}, "superEntity": "string", "superEntityStrategy": "string"
  }
}
```

> **⚠️ `_conf` é OBRIGATÓRIO e semântico aqui.** A definição de **entity** exige `_conf` (configuração:
> chave única, índices, controle de concorrência, superentidade…). A convenção "chave `_`-prefixada =
> metadado removido" vale **apenas para o `.model.json`** (publicação de modelo), **não** para a
> definição de entity. **Não** remova `_conf` (nem outras chaves) deste payload — é outro contrato, em
> outro endpoint. Ver [model-format — chaves de metadado](../../persistence-crs/spec/model-format.md#chaves-de-metadado-_-prefixadas).

## Colunas obrigatórias de toda projeção — `aggregateid`, `status` e cada `whenAttribute` de evento (REGRA)

Além das colunas derivadas de `data.attribute` + valueObjects, **toda entity que é projeção de um
agregado DEVE declarar** as colunas de projeção abaixo — em **todo** agregado, sem exceção:

- **`aggregateid`** — `String`, `length` **36**, `nullable: false`: o **UUID do agregado**; **identifica
  a linha** da projeção (ver [identificação na projeção](../../persistence-crs/spec/model-format.md#identificação-na-projeção-leitura)).
- **`status`** — `String`, `nullable: false`: o **estado atual** do agregado (campo de filtro típico em
  [persistence-q](../../persistence-q/README.md)).
- **cada `whenAttribute` de evento** — `Timestamp`, `nullable: true`: o **carimbo de tempo do evento**
  (ex.: o evento `criada` tem `whenAttribute` `criadaem`). Há **uma coluna por evento** do agregado. O
  valor é **preenchido automaticamente pela plataforma na escrita** (por isso `whenAttribute` **NÃO** é
  um `data.attribute` de comando no `.model.json` — não o declare lá), mas a **coluna na projeção
  precisa existir**: ela **não** deriva de `data.attribute` e **não** é auto-injetada.

**Não** são reservados (§reservados acima) **nem** auto-injetados pela plataforma — o Forger injeta
automaticamente **apenas** `id`/`logversion`/`logrole`/`loguser`. `aggregateid`, `status` e **cada
`whenAttribute` de evento** são **declarados pelo autor da entity**, obrigatoriamente. **Faltando
qualquer um deles, a projeção do agregado não é materializável**: o fluxo de projeção do es-n grava a
linha por `aggregateid`, registra `status` e escreve o carimbo em cada `whenAttribute` — se a coluna
correspondente não existir, o consumidor de projeção falha (atributo/componente desconhecido) e a linha
**nunca** é materializada (consultas em [persistence-q](../../persistence-q/README.md) retornam vazio).

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
```

Vale igual para atributos e associações: declarar um atributo só com `nullable` altera **apenas**
`nullable` — `length`, `unique` e `comment` continuam como estavam.

Três detalhes que costumam morder:

- **`null` explícito conta como ausente.** Para limpar, use o vazio do tipo (`[]`, `""`), não `null`.
- **`accessControl` é substituído por inteiro**, não mesclado chave a chave: enviar só `read` reseta
  `write` para `["MASTER"]`. Não existe entity sem controle de acesso — `MASTER` é o piso.
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
