# es-n — guia do serviço

> **Papel:** daemon de event sourcing. **Escuta** cada mudança de estado de agregado, **reconstitui** o
> agregado em JSON e o **despacha** para filas — atualizando projeções e disparando coordenação.
> É majoritariamente **autônomo**: um agente que constrói sistemas raramente o chama; precisa, porém,
> **entender seu comportamento** para modelar corretamente os eventos.
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Contents
- O que o es-n faz
- Pipeline de eventos
- Os três tipos de despacho
- Ordenação e idempotência
- Endpoint (operacional)
- Pontos de coordenação
- Pitfalls (ao modelar eventos)

---

## O que o es-n faz

Toda vez que um comando bem-sucedido no [persistence-crs](../persistence-crs/README.md) grava um
evento, a mudança de estado **notifica** o es-n (CP-3). O es-n então:

1. lê o(s) evento(s) pendentes com garantia de **processamento exatamente-uma-vez**;
2. **reconstitui o estado atual do agregado** em JSON;
3. **despacha** esse agregado para uma ou mais filas, conforme o **modelo do evento**.

## Pipeline de eventos

```
mudança de estado (persistence-crs grava evento)
        │  notifica (mesma transação)
        ▼
   es-n: lê eventos pendentes ──▶ avança checkpoint (exatamente-uma-vez)
        │
        ▼
   reconstitui agregado (JSON)
        │
        ▼
   despacha para fila(s) do broker de mensagens  ──▶ (ver "três tipos de despacho")
```

O **checkpoint** registra até onde os eventos de cada tenant/tipo de agregado já foram processados.
O avanço só ocorre **após** o despacho bem-sucedido; uma falha **não** avança o checkpoint, garantindo
reprocessamento sem perda e sem duplicação indevida.

**Mecânica (comportamento):** o checkpoint guarda a posição do último evento processado. A leitura de
eventos pendentes reserva o trabalho de forma a permitir **múltiplos processadores concorrentes** sem
que dois processem o mesmo evento. O avanço do checkpoint é **atômico** com o processamento: se o
despacho falha, o checkpoint permanece, e o evento é **reprocessado** numa próxima passagem (replay
seguro). Combinado com a deduplicação na projeção, isso dá **exatamente-uma-vez** na aplicação.

**Envelope do evento:** ao despachar, o es-n monta o **estado atual do agregado** em JSON e o envolve
com os dados necessários ao consumidor (identificador do agregado, identificador do evento, contexto e
tenant, e o carimbo de tempo do evento já formatado). O consumidor recebe esse envelope pronto, sem
precisar reconstituir o agregado.

## Os três tipos de despacho

O **modelo do evento** (no `.model.json` publicado pelo forger) declara para onde o agregado vai. Há
até três destinos, combináveis:

| # | Destino | O que faz | Declarado por |
|---|---|---|---|
| 1 | **Projeção** (comum) | Atualiza a **projeção** do próprio contexto no banco de leitura. | comportamento padrão do evento |
| 2 | **Projeção cross-contexto** | Atualiza a projeção de **outro** contexto/tenant. | marcador de **projeção** (`triggerProjection`) |
| 3 | **Coordenação** | Dispara uma **saga** entre agregados/contextos. | marcador de **coordenação** (`triggerCoordination`) |

- O destino **1** é o caminho comum e usa a [fila de projeção](../coordenacao.md) → aplicada pelo
  persistence-crs (CP-5). A **ordem de entrega** desse destino é escolhida por `event.<endState>.domainBus.orderingMode`:
  `per-aggregate` (**default** — particiona por `hash(aggregateid)`, preservando ordem por instância com paralelismo entre instâncias),
  `strict` (ordem global total, consumidor único) ou `none` (sem garantia de ordem, máxima vazão). Sem o marcador → `per-aggregate`.
- Os destinos **2** e **3** carregam, além dos dados, o **tenant/contexto de destino** e o nome da
  projeção/coordenação alvo.
- O destino **3** é consumido pelo persistence-crs, que aciona o **br-service** (coordenação) e, conforme
  o resultado, submete um **novo comando** — fechando a saga (CP-7).

### Topologia de filas é fixa e global, não por tenant nem dinâmica
Ponto importante para quem modela domínios cross-contexto:

- **Toda** a topologia — filas de **projeção do próprio contexto** (incluindo as variantes de ordenação
  `strict`/`per-aggregate`), de **projeção cross-contexto** e de **coordenação** — é **infraestrutura
  universal, global e fixa** da plataforma. Os nomes de exchange/fila são **fixos** (não interpolam
  tenant): existe **um** conjunto compartilhado por **todos** os tenants.
- Ela é **declarada uma única vez na inicialização** do serviço (idempotente) — **não** há criação
  dinâmica nem por tenant, nem por par de contextos, nem por agregado, nem por evento. (Não existe fila
  por tenant: a declaração é a mesma para todos.)
- O **isolamento por tenant não está na fila** e sim no **conteúdo da mensagem**: no **despacho**, o
  es-n embute no envelope o tenant/contexto de destino (e o alvo — esquema/nome da projeção ou da
  coordenação, declarado no marcador do evento); na **recepção**, o consumidor **decodifica** esse
  envelope e separa dado e fluxo de execução pelo tenant ali contido. Filas compartilhadas, separação
  **por payload** — em ambas as pontas.
- **Consequência para o autor do modelo:** declarar `triggerProjection`/`triggerCoordination` no evento
  **não cria filas** — apenas faz o agregado ser despachado pelos canais universais já existentes, com
  o destino no conteúdo. (Filas declaradas a partir de um diagrama de processo são outra coisa: ver
  [forger/process](../forger/endpoints/process.md).)

## Ordenação e idempotência

- **Ordenação** (por evento, configurável no modelo) — define como as mensagens são distribuídas aos
  consumidores, com trade-off entre **ordem** e **paralelismo**:

  | Modo | Garantia de ordem | Paralelismo | Quando usar |
  |---|---|---|---|
  | **nenhuma** | sem ordem | máximo | eventos independentes, throughput acima de ordem |
  | **estrita** | ordem global (um consumidor único) | mínimo | quando a ordem entre agregados diferentes importa |
  | **por agregado** (padrão) | ordem **dentro de cada agregado** | alto (distribuição por partição entre agregados) | caso geral — ordem onde importa, paralelo onde não importa |

- **Idempotência:** a aplicação da projeção é deduplicada por evento; reentregas não geram efeito duplo.

## Endpoint (operacional)

O es-n expõe **uma** operação de serviço para **acionar o processamento** dos eventos de um tenant
(uso operacional/interno, não é uma API de cliente): ver
[endpoints/processar-eventos.md](endpoints/processar-eventos.md).

## Pontos de coordenação
- **CP-3** — recebe a notificação de mudança de estado do persistence-crs.
- **CP-4** — despacha o agregado para até três destinos (projeção, projeção cross-contexto, coordenação).
Ver [coordenação](../coordenacao.md).

## Pitfalls (ao modelar eventos)

- [ ] Definir a **ordenação** adequada por evento: use **por agregado** (padrão) salvo necessidade de
      ordem global estrita (que reduz paralelismo).
- [ ] Declarar **projeção cross-contexto** e **coordenação** só quando necessário — cada marcador gera
      um despacho adicional.
- [ ] Lembrar que a projeção é **assíncrona**: não há garantia de leitura imediata após o comando.
- [ ] Tornar processadores de coordenação/projeção **idempotentes** — pode haver reentrega.
