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

## Comando com data

```
POST /a/vendas/pedido
{ "agendar": { "id": "<uuid>", "status": "criada",
               "entregaem": "2026-09-15T14:00:00-03:00" } }    // 14h em Brasília

→ 200 { "id": "<uuid>", "status": "agendada" }

GET /a/vendas/pedido/<uuid>
→ 200 { ..., "entregaem": "2026-09-15T17:00:00", "agendadaem": "2026-09-13T20:19:09" }
```

O `entregaem` voltou em **UTC** (17h), sem fuso — é a forma em que foi gravado, e a mesma que a projeção
devolve. O `agendadaem` é o carimbo automático do evento, também em UTC. Ver
[endpoints/comando.md § Campos de data](endpoints/comando.md).

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
