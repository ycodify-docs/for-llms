# es-n · catálogo de erros

> O es-n é um daemon; a maioria das falhas é tratada internamente, classificada e publicada de forma
> assíncrona para o subsistema de monitoração. Guia: [README.md](README.md).

## Respostas do endpoint operacional

| HTTP | Significado | Correção |
|---|---|---|
| `400` | `tenantId` inválido (ausente ou não-UUID). | Enviar um identificador de tenant válido. |
| `503` | Subsistema de assinatura/checkpoint indisponível. | Aguardar/retentar; verificar a saúde do serviço. |
| `500` | Falha ao processar eventos do tenant. | Retentar; o checkpoint não avança em falha. |

## Categorias de falha interna (diagnóstico)

Agrupadas por etapa do pipeline (sem expor detalhes de infraestrutura):

| Etapa | Categoria | Significado |
|---|---|---|
| Assinatura/notificação | Processador de assinatura; processamento de notificação | Falha ao escutar/receber mudanças de estado. |
| Checkpoint | Trava de checkpoint; repositório de assinatura | Falha ao reservar/avançar o ponto de processamento. |
| Leitura de evento | Leitura; conversão (parse) de evento | Falha ao ler/interpretar o evento de origem. |
| Modelo | Leitura de cache; descoberta de tenant; parse de modelo | Falha ao obter/interpretar o modelo do agregado. |
| Despacho | Publicação em fila; inicialização da topologia; saúde da mensageria | Falha ao despachar para o broker de mensagens. |
| Cliente downstream | Chamada a serviço upstream | Falha ao consultar outro serviço necessário. |
| Endpoint | Entrada inválida no acionamento | `tenantId` inválido. |
| Desconhecida | Não classificada | Escalar com contexto. |

## Nota
Como o checkpoint só avança após despacho bem-sucedido, falhas transitórias resultam em
**reprocessamento** seguro (com idempotência na aplicação da projeção), não em perda de eventos.
