# persistence-crs · exemplos anotados

> Dados genéricos (`acme.vendas`, agregado `pedido`). Cabeçalhos `Authorization` + `X-Tenant-Id` em todas.

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.
> Guia: [README.md](README.md).

## Executar um comando

```
POST /a/vendas/pedido
Headers: Authorization: ..., X-Tenant-Id: <tenant-id>
{ "criar": { "cliente": "C-1", "total": 25000 } }      // total em centavos (tipo Long)

→ 200 { "id": "<uuid do agregado>", "status": "criada" }
```

O comando `criar`, seus campos e o estado `criada` vêm do **model** publicado (gramática:
[spec/model-format.md](spec/model-format.md); exemplos: [examples/](../examples/README.md)).

## Comando que falha por transição

```
POST /a/vendas/pedido
{ "faturar": { } }              // pedido ainda em "criada"; faturar exige "confirmada"

→ 510  (falha de transição de estado)
```

Correção: enviar antes o comando que leva ao estado de origem exigido (ex.: `confirmar`).

## Ler estado atual

```
GET /a/vendas/pedido/<uuid>
→ 200 { ...estado atual do agregado... }
```

## Ler histórico

```
GET /a/vendas/pedido/<uuid>/history
→ 200 [ {evento1}, {evento2}, ... ]
```

## CRUD de entidade convencional (`POST /e`)

Entidade `contato` — tabela simples (**não** é projeção de agregado). O verbo vem do campo `action`; a
chave primária `id` é gerada na criação e identifica a linha em update/delete.

```
POST /e
Headers: Authorization: ..., X-Tenant-Id: <tenant-id>
{ "action": "CREATE", "data": [ { "nome": "Ana", "email": "ana@ex" } ] }   // id é gerado; omita

→ 201 [ { "id": 1, "nome": "Ana", "email": "ana@ex" } ]
```

```
POST /e
{ "action": "UPDATE", "data": [ { "id": 1, "email": "ana@novo" } ] }   // id localiza a linha

→ 200   (vazio)
```

```
POST /e
{ "action": "DELETE", "data": [ { "id": 1 } ] }

→ 200   (vazio)
```

Lote (várias operações numa chamada, aplicadas na ordem):

```
POST /e
[ { "action": "CREATE", "data": [ { "nome": "Beto" } ] },
  { "action": "DELETE", "data": [ { "id": 2 } ] } ]

→ 200   (vazio)
```

Detalhe do contrato: [endpoints/entidade.md](endpoints/entidade.md).

## Observação sobre leitura após comando
Depois de um comando bem-sucedido, **consultar a projeção** ([persistence-q](../persistence-q/README.md))
pode levar um curto intervalo até refletir a mudança — a projeção é atualizada de forma assíncrona
(ver [arquitetura](../01-arquitetura.md)).
