# persistence-crs · endpoints · escrita de projeção

> Escrita direta numa **projeção** (criar/atualizar/remover linhas no banco de leitura). Endpoint de
> **serviço autorizado** — no fluxo normal, projeções são mantidas de forma assíncrona pelo caminho
> es-n → projeção (CP-5); este endpoint é o ponto de aplicação dessa escrita. Guia: [../README.md](../README.md).

## Requisição

`POST /e`

Cabeçalhos: `Authorization` + cabeçalho de tenant `X-Tenant-Id`.

Corpo (operação única):

```json
{ "action": "CREATE | UPDATE | DELETE", "data": [ { "...": "..." } ] }
```

Corpo (lote): um **array** de operações no formato acima.

## Respostas

| Ação | Sucesso |
|---|---|
| `CREATE` | `201` (devolve os dados criados) |
| `UPDATE` | `200` |
| `DELETE` | `200` |

Ação desconhecida → `400`. Falha de aplicação → `510`.

## Comportamento
A operação é aplicada à projeção do tenant no banco de leitura. No fluxo dirigido por eventos, a
aplicação é **idempotente** (deduplicação por evento) e pode ser **ordenada** (nenhuma/estrita/por
agregado), conforme o despacho do [es-n](../../es-n/README.md).

## Erros
`400` (ação/corpo inválido), `403` (tenant), `510` (falha de aplicação — ex.: violação de restrição).
Catálogo: [../erros.md](../erros.md).
