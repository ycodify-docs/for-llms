# filer · endpoints · remover

> Remove um arquivo específico **ou** todos os que casam com um prefixo (lote). Guia: [../README.md](../README.md).

## Requisição
`DELETE /api/file/delete`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id` |
| Parâmetro `filename` | query (ou `x-filename`): chave **completa** (um arquivo) **ou** **prefixo parcial** (lote) |

Prefixos (lote): `…-{entity}-{entityId}-{attribute}` (por atributo), `…-{entity}-{entityId}-` (todos do
agregado), `…-{entity}-` (todos do tipo).

## Resposta
`200` — relatório JSON:
```json
{ "deleted": 3, "prefix": "<referência>", "files": [ "<ref1>", "<ref2>", "<ref3>" ] }
```
Em padrão de lote sem correspondências, `deleted: 0` (ainda `200`).

## Autorização
Requer permissão de **escrita** no agregado (entity).

## Erros
| HTTP | Quando |
|---|---|
| `400` | chave/prefixo ausente ou inválido |
| `401` | token inválido |
| `403` | sem permissão de escrita |
| `404` | arquivo **específico** não encontrado (não se aplica a lote) |
| `500` | falha de armazenamento |

Catálogo: [../erros.md](../erros.md).
