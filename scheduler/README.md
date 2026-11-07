# scheduler — agendamento

> **Papel:** guarda uma coordenação declarada com **`schedule`** e a despacha **na hora marcada**. É o que deixa o
> próprio domínio agir numa data — vencer um contrato, lembrar um cliente trinta dias depois, encerrar uma etapa
> parada — sem que ninguém abra o agregado e sem conta de serviço.
> **Interno:** roda dentro da persistência, **sem rota nem endpoint público**. Você não o chama: você **declara o
> prazo no modelo**, e a plataforma agenda.
> Pré-requisitos: [model-format — eventos e `domainBus`](../persistence-crs/spec/model-format.md#eventos-e-domainbus),
> [contexto de coordenação do br-service](../br-service/contextos.md#2--coordenação--saga-assíncrono).

## Quando usar

| Use `schedule` para… | Não use para… |
|---|---|
| uma ação que o **tempo** dispara: vencimento, expiração, lembrete, prazo de etapa | uma reação imediata — sem `schedule`, a coordenação já sai logo depois do evento |
| um prazo que **nasce de um dado do agregado** (`expiraem`) ou de um intervalo fixo (`P30D`) | repetir algo periodicamente — não há recorrência; encadeie (ver adiante) |
| uma decisão que deve olhar o **estado na hora** | precisão de segundo — o despacho sai **alguns segundos** depois do instante |

## Como declarar

O prazo é uma propriedade do **item de `triggerCoordination`** do evento — o mesmo item que diz o `name`, o
`targetTenantId` e a rota do processor. Uma chave só, dentro de `schedule`:

| Chave | Valor | O instante do despacho |
|---|---|---|
| `atAttribute` | o nome de um atributo do **evento**, do tipo `Timestamp` ou `Date` | o valor desse atributo |
| `after` | uma duração ISO-8601: `PT2H`, `PT720H`, `P30D` | o carimbo do evento (`whenAttribute`) **mais** a duração |

### Exemplo: o contrato vence sozinho

O contrato nasce `ativo` com uma data de vencimento. Na data, ele passa a `expirado` — a menos que tenha sido
renovado antes.

```jsonc
"contratos.contrato": {
  "command": {
    "ativar": {
      "data": { "attribute": {
        "parte":    { "type": "String",    "length": 120, "nullable": false },
        "expiraem": { "type": "Timestamp", "nullable": false }
      } },
      "fromState": [], "endState": "ativado", "roles": ["ADMIN"]
    },
    "renovar": {
      "data": { "attribute": {
        "status":   { "type": "String",    "length": 20, "nullable": false },
        "expiraem": { "type": "Timestamp", "nullable": false }
      } },
      "fromState": ["ativado", "renovado"], "endState": "renovado", "roles": ["ADMIN"]
    },
    "expirar": {
      "data": { "attribute": { "status": { "type": "String", "length": 20, "nullable": false } } },
      "fromState": ["ativado", "renovado"], "endState": "expirado", "roles": ["ADMIN"]
    }
  },
  "event": {
    "ativado":  { "type": "ativado",  "whenAttribute": "ativadoem",  "payloadInherit": "command-data",
                  "domainBus": { "triggerCoordination": [ {
                    "name": "vencimento", "targetTenantId": "<tenant-id>",
                    "br": { "route": "contratos/verificarvencimento" },
                    "schedule": { "atAttribute": "expiraem" } } ] } },
    "renovado": { "type": "renovado", "whenAttribute": "renovadoem", "payloadInherit": "command-data",
                  "domainBus": { "triggerCoordination": [ {
                    "name": "vencimento", "targetTenantId": "<tenant-id>",
                    "br": { "route": "contratos/verificarvencimento" },
                    "schedule": { "atAttribute": "expiraem" } } ] } },
    "expirado": { "type": "expirado", "whenAttribute": "expiradoem", "payloadInherit": "command-data" }
  }
}
```

Cada `ativar` e cada `renovar` agenda um despacho para o `expiraem` que levou. Renovar **não cancela** o despacho
anterior: ele acorda na data antiga, o processor vê que o contrato foi renovado para depois, e não faz nada. Quem
decide é sempre o processor, olhando o estado — ver [como escrever o processor](#como-escrever-o-processor).

## O que vale em cada caso

| O item declara | O valor no evento | O que acontece |
|---|---|---|
| nada (`schedule` ausente) | — | sai **agora**, como sempre |
| `atAttribute` | `Timestamp` no futuro (`2026-12-01T03:00:00`, UTC) | sai nesse instante |
| `atAttribute` | `Date` (`2026-12-01`) | sai às **00:00 UTC** desse dia |
| `atAttribute` | instante que já passou, ou agora | sai **agora** |
| `atAttribute` | atributo **ausente, nulo ou vazio** no evento | **não sai**, nem agora nem depois — a coordenação dependia da data |
| `after` | — | sai no carimbo do evento mais a duração; se isso já passou, sai agora |

**Recusas — a coordenação não sai, e a falha fica registrada para o suporte:**

- `atAttribute` e `after` juntos, ou `schedule` vazio;
- chave desconhecida dentro de `schedule` (um erro de digitação como `atAtribute`);
- `after` que não é duração ISO-8601, ou negativa;
- valor do atributo que não é data nem instante.

> ⚠️ **A publicação do modelo não confere `schedule`.** Um `schedule` mal escrito é aceito na publicação, o comando
> responde normalmente, e a recusa só aparece **no despacho**, sem chegar a quem enviou o comando. Confira a forma
> contra o [schema](../persistence-crs/spec/model.schema.json) (`$defs.schedule`) antes de publicar.

> ⚠️ **`Date` vence à meia-noite UTC**, que é 21h do dia anterior em Brasília. Se o domínio entende "vence no fim
> do dia 1º", use um `Timestamp` com a hora certa, ou faça o processor comparar do jeito que o domínio entende.

## Na hora marcada

O despacho agendado entra **no mesmo caminho** da coordenação imediata — nada muda do lado do processor:

- o processor recebe o **evento original** (os dados como estavam quando o evento foi gravado), o
  **`aggregateState` lido naquele instante**, o `targetTenantId` e o `sourceTenantId`
  ([forma exata](../br-service/contextos.md#2--coordenação--saga-assíncrono));
- **não há token**. O comando que o processor devolver é executado com a identidade de quem fez o comando de
  origem, e o papel dela não é reavaliado — como em toda coordenação
  ([por onde a identidade passa](../persistence-crs/spec/model-format.md#por-onde-a-identidade-passa-na-coordenação));
- vale a mesma deduplicação por (evento, `name`): o mesmo despacho não executa o comando-alvo duas vezes.

## Como escrever o processor

**Decida pelo estado de agora, não pelo evento de antes.** Entre o agendamento e o despacho o agregado pode ter
mudado: renovado, cancelado, expirado por outro caminho. O `aggregateState` traz o estado desse instante.

```
função(data):
    contrato = data.aggregateState
    se contrato.status não está em ["ativado", "renovado"]:  retorna {}        // já saiu do caminho: nada a fazer
    se contrato.expiraem > agora_em_UTC():                    retorna {}        // foi renovado para depois
    retorna { processedData: { targetCommand: {
        boundedContext: "contratos", aggregateType: "contrato",
        commandName: "expirar", data: { id: data.contrato.aggregateid, status: contrato.status } } } }
```

- **Não devolver `targetCommand` é o jeito certo de não fazer nada** — é o que torna inofensivo o despacho de uma
  data que deixou de valer.
- **Seja idempotente.** O despacho é entregue **pelo menos uma vez**: numa queda no meio do caminho, o mesmo
  processor pode ser chamado de novo. Conferir o estado, como acima, já resolve.
- **`aggregateState` pode chegar `{}`** quando o agregado de origem não é conhecido no tenant de destino
  ([detalhe](../br-service/contextos.md#2--coordenação--saga-assíncrono)). Com `{}` não há como conferir o estado:
  para prazo, prefira coordenar **no mesmo tenant** do agregado.
- **Compare datas em UTC.** `Timestamp` é gravado e devolvido em UTC, sem fuso
  ([datas](../persistence-crs/spec/model-format.md#datas-timestamp-e-date)).

## Garantias e limites

| | |
|---|---|
| **Durabilidade** | o agendamento fica guardado e sobrevive a reinício da plataforma |
| **Entrega** | pelo menos uma vez, com retentativa espaçada se o despacho falhar; a deduplicação por (evento, `name`) impede o comando-alvo de rodar duas vezes |
| **Precisão** | alguns segundos depois do instante marcado |
| **Gravar o agendamento** | acontece quando o evento é despachado. Se não puder ser gravado nesse momento, há poucas retentativas; esgotadas, a coordenação **não sai** e fica registrada como pendente para o suporte, **sem reentrega automática** |
| **Cancelar** | não há cancelamento pelo modelo: o processor decide, na hora, que não há nada a fazer |
| **Repetir** | não há recorrência. Para um ciclo, encadeie: o comando-alvo grava um evento que agenda o próximo |
| **Um alvo** | como em toda coordenação, um `targetCommand` por despacho ([detalhe](../persistence-crs/spec/model-format.md#o-que-o-processor-de-coordenação-devolve)) |
