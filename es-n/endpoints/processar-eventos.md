# es-n · endpoints · processar eventos de um tenant

> Operação de **serviço/operacional** que aciona o processamento dos eventos pendentes de um tenant
> (por exemplo, para incorporar um tenant ao processamento sem reinício, ou forçar um ciclo). **Não** é
> uma API de cliente final. Guia: [../README.md](../README.md).

## Requisição

`POST /api/event-processor/tenants/{tenantId}/process`

- `{tenantId}` — identificador do tenant (UUID). Único parâmetro; sem corpo.

## Respostas

| HTTP | Significado |
|---|---|
| `200` | Processamento acionado para o tenant. |
| `400` | `tenantId` ausente/vazio ou em formato inválido (não-UUID). |
| `503` | Subsistema de assinatura/checkpoint indisponível no momento. |
| `500` | Falha ao processar os eventos do tenant. |

Corpo de resposta: objeto com `status` e `message` (e o `tenantId` em caso de sucesso).

## Comportamento

Dispara o [pipeline de eventos](../README.md#pipeline-de-eventos) para o tenant: leitura dos eventos
pendentes a partir do checkpoint, reconstituição do agregado e despacho. O avanço do checkpoint é
**exatamente-uma-vez**; uma falha não avança o checkpoint (haverá reprocessamento seguro).
