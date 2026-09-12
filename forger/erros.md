# forger · catálogo de erros

> Envelope e códigos de erro do serviço. Guia: [README.md](README.md).

## Envelope

Em erro, a resposta tem o status HTTP correspondente e um corpo JSON com esta forma:

```json
{
  "timestamp": 1757520000000,
  "orgName": "acme",
  "projectName": "vendas",
  "endpoint": "/org/acme/project/vendas/...",
  "trace": null,
  "statusCode": "400",
  "default-message": "Requisição inválida.",
  "message": "DataSchema 'pedidos' não pode voltar a RUNNING: ...",
  "cause": "..."
}
```

> **Leia a `message`, não a `default-message`.** A `default-message` é um texto fixo por classe de erro,
> em pt-BR; a **`message`** é a explicação específica daquela falha — é ela que diz *qual* campo faltou
> ou *o que* fazer. O `statusCode` vem como **string** e repete o status HTTP.
>
> `cause` só aparece quando a exceção tem causa encadeada, e `trace` só é preenchido quando o rastreio
> está ligado na instância; nos demais casos vem `null`. `orgName`, `projectName` e `endpoint` são
> preenchidos nos endpoints que os conhecem.

(Em casos específicos o corpo traz campos adicionais — ex.: publicação de **process** com problemas
críticos retorna também o relatório de análise.)

## Códigos

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `400` | Requisição inválida | Campo obrigatório ausente (`_conf`, `logversion`); `type` de atributo fora da lista fechada ou em minúsculas; valor fora do formato (nome em minúsculas, comprimento, porta > 0); documento de modelo sem seção de agregado; **volta a `RUNNING` sem o `.model.json` publicado**; **projeção que não confere com o modelo de escrita** (ver abaixo). | Corrigir o corpo conforme o contrato do endpoint. No caso da transição, republicar o modelo — ver [dataschema](endpoints/dataschema.md#alterar-o-schema-de-um-sistema-em-operação). |
| `403` | Acesso negado | Sem papel de administrador/engenheiro na organização; inconsistência de propriedade (`org`/`project`/`tenant`). | Usar credencial autorizada; conferir que o tenant pertence ao contexto. |
| `404` | Não encontrado | Recurso inexistente; `tenant` não referencia dataschema do contexto. | Verificar a ordem de deploy e os identificadores. |
| `409` | Conflito de versão | `logversion` divergente do atual (alguém atualizou antes); ou remoção bloqueada por dependentes. | Reler o recurso, reenviar com o `logversion` atual; remover dependentes antes. |
| `422` | Não processável | Publicação de **process** com problemas **críticos** na análise. | Corrigir o processo conforme o relatório retornado. |
| `500` | Erro interno | Falha na compilação/aplicação de DDL; falha ao materializar efeito físico (rollback aplicado). | Revisar a definição; retentar; se persistir, escalar. |

## Projeção que não confere com o modelo de escrita — `400` na volta a `RUNNING`

Vale para entity que declara [`_conf.projectionOf`](endpoints/entity.md#declarar-que-a-entity-é-projeção--_confprojectionof-regra).
A recusa acumula **todos** os achados numa resposta só, **nada é gravado** e o dataschema **continua em
`MODELING`**.

| O que motiva | O que dizer ao corrigir |
|---|---|
| **atributo acima do teto** — a entity declara o que agregado nenhum escreve | remover o atributo da entity, **ou** acrescentá-lo a um `command` do agregado e republicar o `.model.json` |
| **coluna faltando (piso)** — os agregados escrevem o que a entity não declara, carimbos de evento incluídos | declarar os atributos na entity, **ou** remover do `.model.json` o que a projeção não deve receber. É o caso grave: sem a coluna a gravação inteira é recusada e a linha nunca materializa |
| **agregado inexistente** — nome em `projectionOf` sem `type` correspondente | corrigir o nome (é o **`type`**, não a chave composta do mapa `aggregate`), ou republicar o modelo com esse agregado |
| **`type` ambíguo** — dois agregados publicados com o mesmo `type` | dar `type` distintos e republicar o modelo |
| **chave divergente** — `identity.fields` de um campo sem `unique` no atributo, ou de 2+ campos sem `_conf.uniqueKey` igual, ou `unique` sem lastro nenhum | declarar a chave no lugar certo (um campo → atributo `unique`; dois ou mais → `_conf.uniqueKey`), **ou** acertar o `identity.fields` e republicar o `.model.json` |
| **`aggregateid` ou `status` não declarados** na projeção | declarar os atributos na entity (é um caso particular do piso, e também é recusado na criação/atualização da entity) |
| **modelo publicado ilegível** | republicar o `.model.json` |

Na criação/atualização da entity, o `400` sai antes, e por outras causas: `projectionOf` em `_conf.type`
diferente de `entity`, projeção sem `aggregateid`/`status`, item vazio ou repetido.

## Notas de comportamento
- **Rollback:** ao criar `database`/`dataschema`/`entity`, se o efeito físico falhar, os metadados são
  revertidos — o recurso não fica num estado parcial.
- **Remoção bloqueada:** qualquer recurso com dependentes não pode ser removido (cadeia
  dbconn ⟶ database ⟶ dataschema ⟶ entity; project ⟶ dataschema).
