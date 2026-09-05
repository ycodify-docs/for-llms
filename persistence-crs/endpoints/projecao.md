# persistence-crs · endpoints · escrita de projeção

> Escrita direta numa **projeção** (criar/atualizar/remover linhas no banco de leitura). Guia:
> [../README.md](../README.md).
>
> **⚠️ Esta rota NÃO é o caminho do fluxo por evento.** No fluxo normal (es-n → projeção, CP-5) a linha é
> gravada por um caminho **interno ao serviço**, que não passa por esta rota nem por requisição externa
> alguma — e, por ser interno, não usa a autorização descrita abaixo. O que está documentado aqui é a
> porta **externa**: o que você pode chamar.
>
> **⚠️ E esta rota não é um endpoint de serviço:** é o **mesmo** `POST /e` de [entidade](entidade.md),
> com a mesma e única autorização — pertencer ao tenant e ter papel em `_conf.accessControl.write`. Não
> existe credencial de serviço. Qualquer usuário final que satisfaça esses dois pontos escreve aqui, em
> **qualquer linha**, pulando o comando, o RBAC de comando e o processor `br`. Leia a advertência de
> [entidade](entidade.md) antes de usar.

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
A operação é aplicada à projeção do tenant no banco de leitura, na requisição.

**Idempotência e ordenação são do [es-n](../../es-n/README.md) e do caminho interno, não desta rota.** No
fluxo dirigido por eventos a aplicação é deduplicada por evento e pode ser ordenada (nenhuma/estrita/por
agregado), conforme o despacho do es-n. Uma chamada `POST /e` **não carrega identificador de evento
algum** — não há por onde deduplicar, e reenviar aplica de novo.

## Erros
`400` (ação/corpo inválido), `403` (tenant), `510` (falha de aplicação — ex.: violação de restrição).
Catálogo: [../erros.md](../erros.md).
