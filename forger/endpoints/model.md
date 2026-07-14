# forger · endpoints · model

> **model** = o **modelo de domínio** (documento declarativo) que descreve agregados, comandos, eventos
> e suas transições para um bounded context. Publicá-lo o disponibiliza no **cache distribuído** para os
> demais serviços. Guia: [../README.md](../README.md). Forma do documento: ver [examples/](../../examples/README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Publicar

`POST /org/{org}/project/{project}/tenant/{tenantId}/model`

Envio: o documento de modelo (`.json`) como **upload `multipart/form-data`, no campo `file`** (ex.:
`curl -F "file=@modelo.json"`). **`Content-Type: application/json` (JSON no corpo) → `415`**. Efeito: valida a consistência
`(org, project, tenant)` — o `tenantId` deve referenciar exatamente um dataschema do contexto — e
**publica** o modelo no cache distribuído. **Semântica de sobrescrita**: republicar substitui o anterior.
Resposta `201`: `{ "key": "<referência de publicação>" }`.
Erros: `400` (documento inválido / faltando seção de agregado), `403`/`404` (consistência), `500`.

## Ler

`GET /org/{org}/project/{project}/tenant/{tenantId}/model` → `200` com o conteúdo do modelo; `404` se
não publicado.

## Listar (por contexto)

`GET /org/{org}/project/{project}/model` → `200` array de
`{ tenantId, dataschema, project, status, ... }`.

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

## Coordenação
**CP-1:** o modelo publicado aqui é a **fonte da verdade** lida por **persistence-crs** (quais comandos/
transições existem, quando chamar regra/coordenação), **es-n** (marcadores de despacho do evento) e
**persistence-q** (quais projeções existem). Publique **após** criar as entities.
