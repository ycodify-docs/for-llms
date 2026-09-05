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

## Observação sobre leitura após comando
Depois de um comando bem-sucedido, **consultar a projeção** ([persistence-q](../persistence-q/README.md))
pode levar um curto intervalo até refletir a mudança — a projeção é atualizada de forma assíncrona
(ver [arquitetura](../01-arquitetura.md)).
