# persistence-crs · endpoints · leitura de agregado

> Leitura do **estado atual** e do **histórico de eventos** de um agregado, direto pelo lado de escrita.
> Para consultas por critério sobre projeções, use [persistence-q](../../persistence-q/README.md).
> Guia: [../README.md](../README.md).

Cabeçalhos: `Authorization` + cabeçalho de tenant `X-Tenant-Id`.

## Estado atual

`GET /a/{boundedContext}/{aggregateType}/{id}`

- `200` — estado atual do agregado (JSON), reconstituído a partir dos seus eventos.
- `204` — agregado não encontrado.

## Histórico de eventos

`GET /a/{boundedContext}/{aggregateType}/{id}/history`

- `200` — lista (array) dos eventos do agregado, em ordem.
- `204` — sem eventos.

> **`{id}` = UUID do agregado** (o `aggregateid` da projeção; `projecao.aggregateid == aggregate.id`).
> **Não** use a PK `id` (Long) da projeção — enviar o Long → `510` "Invalid UUID string".

## Quando usar

- **Estado atual:** inspeção pontual de um agregado conhecido pelo seu `id` (UUID do agregado).
- **Histórico:** auditoria/depuração da sequência de eventos (event sourcing).
- Para **buscar** por atributos ou listar muitos registros, use a **consulta** de projeções em
  [persistence-q](../../persistence-q/README.md).

## Erros
`403` (tenant não autorizado), `510` (falha de processamento). Catálogo: [../erros.md](../erros.md).
