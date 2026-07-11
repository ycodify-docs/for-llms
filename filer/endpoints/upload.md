# filer · endpoints · upload

> Envia (cria/substitui) um arquivo anexo a um agregado. Guia: [../README.md](../README.md).

## Requisição
`POST /api/file/upload`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id`, `Content-Type: application/octet-stream` |
| Parâmetro `filename` | query (ou cabeçalho `x-filename`): chave **completa** `{org}-{project}-{entity}-{entityId}-{attribute}.{ext}` |
| Corpo | conteúdo **binário** do arquivo |

## Resposta
`200` — confirmação (texto) com org, project, entity, entityId, attribute e tamanho.

## Autorização
Requer permissão de **escrita** no agregado (entity) conforme o controle de acesso do modelo do tenant.

## Erros
| HTTP | Quando |
|---|---|
| `400` | chave ausente/fora do padrão; extensão não permitida; corpo vazio |
| `401` | token inválido/ausente |
| `403` | sem permissão de escrita no agregado |
| `413` | arquivo acima do limite (≈10 MB) |
| `404` | — |
| `500` | falha de armazenamento |

Catálogo: [../erros.md](../erros.md).

## Comportamento
Segue o [ciclo de vida](../README.md#ciclo-de-vida-uploaddownload). Reenviar a mesma chave **substitui**
o arquivo. `entityId` = id do agregado (`aggregateid`).
