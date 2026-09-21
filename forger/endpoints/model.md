# forger · endpoints · model

> **model** = o **modelo de domínio** (documento declarativo) que descreve agregados, comandos, eventos
> e suas transições para um bounded context. Publicá-lo o disponibiliza no **cache distribuído** para os
> demais serviços. Guia: [../README.md](../README.md). Forma do documento: ver [examples/](../../examples/README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Publicar

`POST /org/{org}/project/{project}/tenant/{tenantId}/model`

Envio **multipart**, com o documento no campo `file` — o arquivo precisa ter extensão `.json`:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  ".../org/acme/project/vendas/tenant/$TENANT_ID/model" \
  -F "file=@pedidos.model.json"
# → 201 { "key": "ENGINE:persistence:cqrs:SETUP-TO:<tenantId>:wm" }
```

Efeito: valida a consistência `(org, project, tenant)` — o `tenantId` deve referenciar exatamente um
dataschema do contexto — e **publica** o modelo no cache distribuído. **Semântica de sobrescrita**:
republicar substitui o anterior, e também cobre o caso de a chave não existir.

Erros: `400` (arquivo vazio, extensão diferente de `.json`, documento inválido), `403`/`404`
(consistência), `500`.

> **A publicação passa a validar o documento contra o metamodelo** e a recusar com `400` o que antes era
> aceito e só falhava — ou silenciava — em runtime. **Em produção desde 2026-09-21** (ver
> [CHANGELOG 1.37](../../CHANGELOG.md)). O `400` traz **todas** as violações de uma vez, cada uma com o
> caminho dentro do JSON. As três que mais aparecem:
>
> | Recusa | Por quê |
> |---|---|
> | chave do agregado ≠ `<boundedContext.name>.<type>` | é o endereço do agregado: o despacho do evento procura **exatamente** essa chave e descarta em silêncio o que não achar, e é a mesma chave que o cliente manda no comando |
> | `command.<cmd>.endState` sem evento de chave igual | o runtime resolve `event[endState]` por chave, sem fallback: o primeiro write falharia com `510` |
> | dois envelopes `<org>.<project>` no mesmo arquivo | só o primeiro seria lido, e o outro sumiria sem aviso |
> | item **objeto** em `domainBus.triggerProjection` sem `targetTenantId` | o destino é descartado no despacho: a projeção cross-tenant simplesmente não dispara. Item **string** continua válido — é a projeção same-tenant |
> | `roles` do aggregate com recorte incoerente | ver [recorte de leitura do aggregate](#recorte-de-leitura-do-aggregate-roles) |

### Recorte de leitura do aggregate: `roles`

**Opcional.** Declara quem lê o aggregate e quais linhas cada papel lê; quem aplica o recorte é o
serviço que atende os endpoints de aggregate. **Aggregate sem a chave publica e lê como antes.**

```json
"roles": {
  "read":   ["OPERADOR", "MASTER"],
  "author": ["OPERADOR"],
  "scope": { "read": { "OPERADOR": { "rows": { "by": "username" } } } }
}
```

Recusas na publicação, todas `400`, iguais às do `accessControl.scope` da entity: papel no `scope.read`
fora de `roles.read`; `MASTER` no `scope`; `rows` sem `by`; `by` em metadado da plataforma (`id`,
`loguser`, `logrole`, `logversion`, `logdate`); `by` em atributo não declarado em comando nenhum do
aggregate.

> ⚠️ **Não confunda com o `roles` do comando**, que é **array**, **obrigatório**, e diz quem pode
> **executar**. Este é **objeto**, **opcional**, e diz quem pode **ler**. O nível desambigua.
>
> Modelo que já estava publicado **não** é revalidado: a checagem acontece na publicação.

### Chaves que um modelo novo não precisa trazer

Estas aparecem em modelos existentes e **nenhum serviço as lê**. Continuam sendo aceitas — não é preciso
reescrever modelo publicado —, mas **não as gere em modelo novo**:

| Chave | Quem decide de fato |
|---|---|
| `schema.forWriteModel.name` | `tenantId.forWriteModel`. O forger **carimba** este campo na publicação e descarta o valor enviado |
| `schema.forReadModel.name` | o **dataschema do tenant** |
| `concurrency.*` | não é lido do modelo; o controle de concorrência vive no `_conf` da **entity** |
| `readProjection` | nada o consome |
| `queue` | filas são provisionadas pelo deploy de **process (BPMN)**, não pelo modelo |

Por agregado, o necessário é: `org`, `project`, `boundedContext`, `type`, `tenantId`, `command` e
`event` — mais `identity`, que é opcional e **é** lido.

> ⚠️ **`boundedContext.name` não é rótulo.** Ele compõe o endereço do agregado **e** nomeia o schema
> onde o log de consumo do evento é gravado. Se divergir do dataschema do tenant, o consumo de evento
> grava no schema de **outro ambiente** sem um único erro em log. Use o mesmo nome nos dois.

## Ler

`GET /org/{org}/project/{project}/tenant/{tenantId}/model` → `200` com o conteúdo do modelo; `404` se
não publicado.

## Listar (por contexto)

`GET /org/{org}/project/{project}/model` → `200` array de
`{ tenantId, dataschema, project, status, cacheKey }`. O `status` é o do **dataschema**, não o do
modelo — serve para ver de relance quais contextos estão em `MODELING` e portanto parados.

## Remover

`DELETE /org/{org}/project/{project}/tenant/{tenantId}/model` → `200` `{ "deleted": true }`.

## Ciclo de vida
Auth → validação de consistência (tenant ↔ dataschema) → **remoção de metadados** (toda chave
`_`-prefixada, em qualquer nível: `_comment`, `_meta`, `_schemaVersion`, …) → validação do documento →
publicação no cache (criar/atualizar) → resposta.

> **Metadados `_`-prefixados são removidos recursivamente** na publicação (não são persistidos no modelo
> nem vistos pelos interpretadores). Use `_comment` para comentário livre. **Não** coloque dado
> semântico sob chave `_`. (Isso é do `.model.json`; a definição de **entity** usa `_conf` obrigatório —
> ver [entity](entity.md).) Gramática: [model-format](../../persistence-crs/spec/model-format.md#chaves-de-metadado-_-prefixadas).

> **Normalização de `schema.forWriteModel.name` na publicação:** o armazém de escrita é **universal**;
> o forger **impõe** o valor canônico — se o `.model.json` enviado não o contiver, ele é **convertido
> automaticamente** (e, se ausente, definido). Não confie no valor enviado nesse campo. Ver
> [model-format](../../persistence-crs/spec/model-format.md#nível-do-agregado).

> **O deploy de `model` NÃO cria filas.** Publicar um `.model.json` apenas valida e o publica no cache
> distribuído. As **filas/topologia de mensageria** são provisionadas pelo deploy de **process (BPMN)**
> — ver [process.md](process.md). O modelo de domínio descreve agregados/comandos/eventos e seus
> marcadores de despacho (projeção/coordenação), mas não declara nem cria infraestrutura de filas.

> ⚠️ **A transição do dataschema para `MODELING` REMOVE o modelo publicado**, e voltar a `RUNNING`
> exige republicá-lo — não é a plataforma que o repõe, porque o `.model.json` é artefato seu e ela não
> guarda outra cópia. Se você remodela entities, este `POST` é passo obrigatório do roteiro, **dentro**
> do bracket: ver [dataschema — alterar o schema de um sistema em operação](dataschema.md#alterar-o-schema-de-um-sistema-em-operação).

## Coordenação
**CP-1:** o modelo publicado aqui é a **fonte da verdade** lida por **persistence-crs** (quais comandos/
transições existem, quando chamar regra/coordenação), **es-n** (marcadores de despacho do evento) e
**persistence-q** (quais projeções existem). Publique **após** criar as entities.
