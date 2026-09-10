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
| `400` | Requisição inválida | Campo obrigatório ausente (`_conf`, `logversion`); `type` de atributo fora da lista fechada ou em minúsculas; valor fora do formato (nome em minúsculas, comprimento, porta > 0); documento de modelo sem seção de agregado; **volta a `RUNNING` sem o `.model.json` publicado**. | Corrigir o corpo conforme o contrato do endpoint. No caso da transição, republicar o modelo — ver [dataschema](endpoints/dataschema.md#alterar-o-schema-de-um-sistema-em-operação). |
| `403` | Acesso negado | Sem papel de administrador/engenheiro na organização; inconsistência de propriedade (`org`/`project`/`tenant`). | Usar credencial autorizada; conferir que o tenant pertence ao contexto. |
| `404` | Não encontrado | Recurso inexistente; `tenant` não referencia dataschema do contexto. | Verificar a ordem de deploy e os identificadores. |
| `409` | Conflito de versão | `logversion` divergente do atual (alguém atualizou antes); ou remoção bloqueada por dependentes. | Reler o recurso, reenviar com o `logversion` atual; remover dependentes antes. |
| `422` | Não processável | Publicação de **process** com problemas **críticos** na análise. | Corrigir o processo conforme o relatório retornado. |
| `500` | Erro interno | Falha na compilação/aplicação de DDL; falha ao materializar efeito físico (rollback aplicado). | Revisar a definição; retentar; se persistir, escalar. |

## Notas de comportamento
- **Rollback:** ao criar `database`/`dataschema`/`entity`, se o efeito físico falhar, os metadados são
  revertidos — o recurso não fica num estado parcial.
- **Remoção bloqueada:** qualquer recurso com dependentes não pode ser removido (cadeia
  dbconn ⟶ database ⟶ dataschema ⟶ entity; project ⟶ dataschema).
