# filer · endpoints · download

> Recupera o conteúdo de um arquivo específico. Guia: [../README.md](../README.md).

## Requisição
`GET /api/file/download`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id` |
| Parâmetro `filename` | query (ou `x-filename`): chave **completa** com extensão `{org}-{project}-{entity}-{entityId}-{attribute}.{ext}` |

## Resposta
`200` — corpo **binário**; `Content-Type: application/octet-stream`; `Content-Disposition: attachment;
filename="{attribute}.{ext}"`.

## Autorização
Requer permissão de **leitura** no agregado (entity).

## Erros
| HTTP | Quando |
|---|---|
| `400` | chave ausente/fora do padrão |
| `401` | token inválido |
| `403` | sem permissão de leitura |
| `404` | arquivo inexistente |
| `500` | falha de armazenamento |

Catálogo: [../erros.md](../erros.md).

## Comportamento
Exige a chave **completa** (com extensão) — diferente de listar/remover, que aceitam prefixo parcial.
