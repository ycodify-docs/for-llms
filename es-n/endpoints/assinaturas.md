# es-n · endpoints · assinatura externa de eventos

> Para quem está **fora** da plataforma e precisa saber **quando** algo acontece num tenant, sem receber
> chamada nenhuma: a conta leitora **lê** um feed por cursor, no ritmo dela. O feed diz **o que** aconteceu e
> **onde**, nunca o conteúdo — o detalhe se lê pelas consultas normais, com o recorte de leitura de quem
> consulta. Guia: [../README.md](../README.md).

> **Estado em 2026-10-01:** **no ar na instância de teste** (`/v3/es/t/…`) desde 04:19:19Z, com a validação da
> declaração na publicação do modelo em produção desde 04:23:36Z. **Na instância de produção (`/v3/es/…`), ainda
> não.** Falta o papel de assinante na gestão de contas para a primeira leitura com conta real.

## Contents
- A assinatura é declarada no modelo
- Onde
- Quem lê
- Ler o feed
- Acompanhar o atraso
- O cursor
- Erros

## A assinatura é declarada no modelo

Não há cadastro por API. A assinatura é parte do `.model.json` do tenant — chave `subscriptions` — e passa a
valer quando o modelo é publicado: dizer o que um sistema de fora consome do domínio é decisão de quem modela.
Forma e regras: [model-format — assinaturas externas](../../persistence-crs/spec/model-format.md#assinaturas-externas-subscriptions).

## Onde

| Superfície | Eventos de |
|---|---|
| `/v3/es/…` | a instância de produção |
| `/v3/es/t/…` | a instância de **teste** |

Como na persistência, **a rota decide a instância**, e cada instância tem o seu armazém de eventos. Um tenant
que escreveu pelas duas rotas tem eventos nas duas, e cada feed vê só os da sua. Os caminhos abaixo são
relativos a essas superfícies.

## Quem lê

Toda requisição leva `Authorization` e `X-Tenant-Id`. Lê **só** a conta que a declaração nomeia em `reader`,
e ela precisa:

- ter o tenant entre os do token;
- ter o papel `SUBSCRIBER` (nome **a confirmar** pela gestão de contas).

Outra conta do mesmo tenant, mesmo com o papel, recebe `403`.

## Ler o feed

`GET /subscriptions/{name}/events?after=<cursor>&limit=<N>`

- `{name}` — o nome da assinatura no modelo.
- `after` — o `next` da resposta anterior. Ausente: o início da assinatura (`from` da declaração, fixado na
  primeira leitura).
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
| `events` | em ordem; **sem conteúdo do evento e sem autoria**. `aggregateType` no nome completo `<org>.<projeto>.<bc>.<tipo>`; `occurredAt` em UTC, sem fuso |
| `next` | onde a próxima leitura começa. **Avança mesmo sem evento do tenant**: um tenant quieto não volta a examinar o que já foi examinado |
| `hasMore` | há mais a ler agora — leia de novo sem esperar |
| `minIntervalMs` | intervalo mínimo entre duas leituras da mesma assinatura |

A resposta é **imediata**: não há espera por evento novo. Com `hasMore: false`, leia de novo depois de
`minIntervalMs` ou mais.

Para ler o detalhe de um evento: [leitura de agregado](../../persistence-crs/endpoints/agregado-leitura.md)
e o histórico dele, com o mesmo token — e com o recorte de leitura que esse token tiver.

## Acompanhar o atraso

`GET /subscriptions/{name}` → `200` com a declaração (`aggregateTypes`, `events`, `reader`) e `lag`:
`delivered` (último `next` entregue), `head` (posição mais recente já legível), `pending` (eventos do filtro
ainda não lidos, contados até 10000; `pendingCapped` diz se o teto foi atingido) e `lastReadAt`.

## O cursor

- **Opaco**: guarde e devolva como veio; não monte nem interprete.
- **Sem perda nem repetição**: o feed só entrega o que já não pode mudar de lugar na ordem — um evento cuja
  gravação ainda está em curso segura a fila até terminar, e nenhum evento aparece depois atrás de um cursor
  já entregue.
- **Reler é permitido**: um `after` antigo devolve de novo o que veio depois dele.
- **Republicar o modelo não perde o cursor**: o filtro novo vale a partir da próxima leitura.

## Erros

| HTTP | Quando |
|---|---|
| `400` | `after` que não é cursor; `limit` fora de 1 a 500 |
| `401` | sem credencial |
| `403` | tenant fora do token; sem o papel `SUBSCRIBER`; conta diferente da que a declaração nomeia |
| `404` | o modelo publicado do tenant não declara a assinatura |
| `429` | leitura antes de `minIntervalMs`; o cabeçalho `Retry-After` diz em quantos segundos |
| `500` | a declaração no modelo está fora da forma — corrija e republique o modelo |

Corpo do erro: `{ "status": <HTTP>, "message": "<motivo>" }`.
