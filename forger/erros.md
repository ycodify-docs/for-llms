# forger · catálogo de erros

> Envelope e códigos de erro do serviço. Guia: [README.md](README.md).

## Envelope

Em erro, a resposta tem o status HTTP correspondente e um corpo JSON:

```json
{ "error": "mensagem" }
```

(Em casos específicos o corpo traz campos adicionais — ex.: publicação de **process** com problemas
críticos retorna também o relatório de análise.)

## Códigos

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `400` | Requisição inválida | Campo obrigatório ausente; valor fora do formato (nome em minúsculas, comprimento, porta > 0); documento de modelo sem seção de agregado. | Corrigir o corpo conforme o contrato do endpoint. |
| `403` | Acesso negado | Sem papel de administrador/engenheiro na organização; inconsistência de propriedade (`org`/`project`/`tenant`). | Usar credencial autorizada; conferir que o tenant pertence ao contexto. |
| `404` | Não encontrado | Recurso inexistente; `tenant` não referencia dataschema do contexto. | Verificar a ordem de deploy e os identificadores. |
| `409` | Conflito de versão | `logversion` divergente do atual (alguém atualizou antes); ou remoção bloqueada por dependentes. | Reler o recurso, reenviar com o `logversion` atual; remover dependentes antes. |
| `422` | Não processável | Publicação de **process** com problemas **críticos** na análise. | Corrigir o processo conforme o relatório retornado. |
| `500` | Erro interno | Falha na compilação/aplicação de DDL; falha ao materializar efeito físico (rollback aplicado); falha ao publicar/remover o modelo no cache na transição de `status` do dataschema (status revertido). | Revisar a definição; retentar; se persistir, escalar. |

## Notas de comportamento
- **`200` que não fez nada:** o `status` do dataschema **não é validado** contra a lista de valores. Um
  erro de digitação (`RUNNIG`) é gravado e responde `200`, mas **não** dispara a publicação/remoção do
  modelo no cache e **não** libera edição de entity — falha silenciosa. Ver
  [dataschema.md](endpoints/dataschema.md#atualizar).
- **Rollback:** ao criar `database`/`dataschema`/`entity`, se o efeito físico falhar, os metadados são
  revertidos — o recurso não fica num estado parcial. Na transição de `status` do dataschema, falha no
  cache também reverte o status (resposta `500`).
- **Remoção bloqueada:** qualquer recurso com dependentes não pode ser removido (cadeia
  dbconn ⟶ database ⟶ dataschema ⟶ entity; project ⟶ dataschema).
