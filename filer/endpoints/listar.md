# filer · endpoints · listar

> Lista os arquivos que casam com um **prefixo** da chave (todos os anexos de um agregado, ou de um
> atributo). Guia: [../README.md](../README.md).

## Requisição
`GET /api/file/list`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id` |
| Parâmetro `filename` | query (ou `x-filename`): **prefixo parcial** (mínimo `{org}-{project}-{entity}-`) |

Exemplos de prefixo:
- `{org}-{project}-{entity}-` → todos os arquivos do tipo de agregado.
- `{org}-{project}-{entity}-{entityId}-` → todos os arquivos **daquele agregado**.
- `{org}-{project}-{entity}-{entityId}-{attribute}` → arquivos daquele **atributo** (qualquer extensão).

## Resposta
`200` — **array JSON** com as referências dos arquivos encontrados; `[]` se nenhum.

## Autorização
Requer permissão de **leitura** no agregado (entity).

## Erros
`400` (prefixo ausente/inválido — menos de 3 partes), `401`, `403`, `500`. Catálogo: [../erros.md](../erros.md).
