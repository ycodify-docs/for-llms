# es-n · endpoints · assinatura externa de eventos

> Para quem está **fora** da plataforma e precisa saber **quando** algo acontece num tenant, sem receber
> chamada nenhuma: o assinante **lê** um feed por cursor, no ritmo dele. O feed diz **o que** aconteceu e
> **onde**, nunca o conteúdo — o detalhe se lê pelas consultas normais, com o recorte de leitura de quem
> consulta. Guia: [../README.md](../README.md).

> **Estado em 2026-10-01: ainda não está no ar.** O serviço está pronto e o armazenamento da instância de
> teste já existe; faltam a implantação, a rota na porta de entrada e o papel de assinante na gestão de
> contas. A instância de produção vem depois da de teste.

## Contents
- Onde
- Autenticação e papéis
- Cadastrar e manter assinaturas
- Ler o feed
- O cursor
- Erros

## Onde

| Superfície | Eventos de |
|---|---|
| `/v3/es/…` | a instância de produção |
| `/v3/es/t/…` | a instância de **teste** |

Como na persistência, **a rota decide a instância**, e cada instância tem o seu armazém de eventos. Um
tenant que escreveu pelas duas rotas tem eventos nas duas, e cada feed vê só os da sua. Os caminhos abaixo
são relativos a essas superfícies.

## Autenticação e papéis

Toda requisição leva `Authorization` e `X-Tenant-Id`. O tenant tem de estar entre os do token.

| Operação | Papel exigido no token |
|---|---|
| cadastrar, listar, ler, alterar e remover assinatura | `ADMINISTRADOR` |
| ler o feed | `SUBSCRIBER` |

> Os nomes dos dois papéis estão **a confirmar** pela gestão de contas. Os papéis valem para todos os
> tenants do token: a conta do assinante deve pertencer a uma organização só.

## Cadastrar e manter assinaturas

`POST /subscriptions`

```json
{
  "name": "monitor-externo",
  "filter": {
    "aggregateTypes": ["acme.vendas.vendas.pedido"],
    "events": ["acme.vendas.vendas.pedido.cancelado"]
  },
  "from": "now"
}
```

| Campo | Regra |
|---|---|
| `name` | de 1 a 80 caracteres — letras, dígitos, espaço, `_`, `.`, `-`; **único no tenant** |
| `filter.aggregateTypes` | ao menos um, até 50; nome completo `<org>.<projeto>.<bc>.<tipo>` |
| `filter.events` | opcional, até 50; nome completo `<org>.<projeto>.<bc>.<tipo>.<evento>`, de um dos tipos acima. Ausente = todos os eventos dos tipos |
| `from` | `now` (padrão): só o que acontecer daqui em diante · `beginning`: desde o primeiro evento |

`201` com a assinatura e `initialCursor`. Nome repetido no tenant: `409`.

| Requisição | Resposta |
|---|---|
| `GET /subscriptions` | `200`, as assinaturas do tenant |
| `GET /subscriptions/{id}` | `200`, a assinatura e o **atraso**: `lag.delivered` (último `next` entregue), `lag.head` (posição mais recente já legível), `lag.pending` (eventos do filtro ainda não lidos, contados até 10000; `lag.pendingCapped` diz se o teto foi atingido), `lag.lastReadAt` |
| `PATCH /subscriptions/{id}` | `200`; aceita só `filter` e `active` |
| `DELETE /subscriptions/{id}` | `204` |

Assinatura com `active: false` continua cadastrada e não pode ser lida.

## Ler o feed

`GET /subscriptions/{id}/events?after=<cursor>&limit=<N>`

- `after` — o `next` da resposta anterior. Ausente: o cursor inicial da assinatura.
- `limit` — de 1 a 500; padrão 100.

```json
{
  "events": [
    {
      "cursor": "v1.MTIzNDo1Ng",
      "context": "vendas",
      "aggregateType": "acme.vendas.vendas.pedido",
      "aggregateId": "00000000-0000-4000-8000-000000000001",
      "event": "cancelado",
      "eventId": 56,
      "occurredAt": "2026-10-01T02:09:33"
    }
  ],
  "next": "v1.MTIzNDo1Ng",
  "hasMore": false,
  "minIntervalMs": 1000
}
```

| Campo | Significado |
|---|---|
| `events` | em ordem; **sem conteúdo do evento e sem autoria**. `occurredAt` em UTC, sem fuso |
| `next` | onde a próxima leitura começa. **Avança mesmo sem evento do tenant**: um tenant quieto não volta a examinar o que já foi examinado |
| `hasMore` | há mais a ler agora — leia de novo sem esperar |
| `minIntervalMs` | intervalo mínimo entre duas leituras da mesma assinatura |

A resposta é **imediata**: não há espera por evento novo. Com `hasMore: false`, leia de novo depois de
`minIntervalMs` ou mais.

Para ler o detalhe de um evento: [leitura de agregado](../../persistence-crs/endpoints/agregado-leitura.md)
e o histórico dele, com o mesmo token — e com o recorte de leitura que esse token tiver.

## O cursor

- **Opaco**: guarde e devolva como veio; não monte nem interprete.
- **Sem perda nem repetição**: o feed só entrega o que já não pode mudar de lugar na ordem — um evento cuja
  gravação ainda está em curso segura a fila até terminar, e nenhum evento aparece depois atrás de um cursor
  já entregue.
- **Reler é permitido**: um `after` antigo devolve de novo o que veio depois dele.

## Erros

| HTTP | Quando |
|---|---|
| `400` | `after` que não é cursor; `limit` fora de 1 a 500; corpo do cadastro fora das regras |
| `401` | sem credencial |
| `403` | tenant fora do token; sem o papel exigido; assinatura de outro tenant; assinatura inativa |
| `404` | assinatura inexistente |
| `409` | nome de assinatura repetido no tenant |
| `429` | leitura antes de `minIntervalMs`; o cabeçalho `Retry-After` diz em quantos segundos |

Corpo do erro: `{ "status": <HTTP>, "message": "<motivo>" }`.
