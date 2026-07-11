# Arquitetura — fluxo ponta a ponta

> Como os oito serviços se conectam: do deploy de recursos à consulta de uma projeção.
> Este documento amarra os fluxos **entre** serviços; o detalhe de cada API está na pasta do serviço.
> Pré-requisitos: [conceitos](02-conceitos.md), [composição](00-plataforma.md).

## Contents
- Visão geral (mapa)
- Fase 0 — Deploy (forger)
- Fase 1 — Comando (síncrona)
- Fase 2 — Notificação e despacho (assíncrona)
- Fase 3 — Projeção
- Fase 4 — Consulta
- Coordenação cross-agregado / cross-contexto

---

## Visão geral (mapa)

```
                          ┌───────────┐
   (1) modelos+recursos   │  forger   │  publica modelos no cache distribuído
   ─────────────────────▶ └─────┬─────┘  cria recursos no banco relacional
                                │
            modelo de domínio publicado (lido pelos demais serviços)
                                │
   cliente                      ▼
   ──── comando ───▶ ┌───────────────────┐   regra/coordenação?  ┌────────────┐
                     │  persistence-crs   │ ────────────────────▶ │ br-service │
                     │  (lado de escrita) │ ◀──────────────────── │ (regra/    │
                     └─────────┬──────────┘   resultado            │  coord.)   │
                               │ grava evento (banco de escrita)   └────────────┘
                               │
                     mudança de estado NOTIFICA
                               ▼
                     ┌───────────────────┐  despacha agregado (JSON) para filas
                     │       es-n         │ ──────────────┬───────────────┐
                     │ (daemon de eventos)│               │               │
                     └───────────────────┘               │               │
                         │ fila de projeção     fila de proj.      fila de
                         ▼                       cross-contexto    coordenação
              ┌────────────────────┐                  │               │
              │  banco de leitura   │ ◀── atualiza ────┘               ▼
              │   (projeções)       │                          (nova coordenação,
              └─────────┬──────────┘                            volta ao crs)
                        │
   cliente              ▼
   ──── consulta ──▶ ┌───────────────────┐
                     │   persistence-q    │  lê projeção materializada
                     │  (lado de leitura) │
                     └───────────────────┘
```

---

## Fase 0 — Deploy (forger)

**Antes do forger, a identidade:** o [orgid](orgid/README.md) cria a **organização** (o `{org}`) e a
**conta dona** com papéis; sem isso o forger não autoriza (CP-10). Estabelecida a identidade, o
**forger** implanta os recursos do sistema, numa ordem obrigatória
(detalhada em [fluxo de deploy](03-fluxo-de-deploy.md)): conexão → banco → contexto+esquema →
projeções (entities) → modelo de domínio → modelo de processo.

Ao final, o **modelo de domínio** está publicado no **cache distribuído** e as tabelas de projeção
existem no **banco de leitura**. Só então os demais serviços têm o que interpretar.

> **Ponto de coordenação:** o modelo publicado por forger é a **fonte** que persistence-crs, es-n e
> persistence-q leem para saber quais agregados, comandos, eventos e projeções existem.

---

## Fase 1 — Comando (síncrona)

1. O cliente envia um **comando** ao **persistence-crs**, identificando bounded context, tipo de
   agregado e o `tenant-id`.
2. O serviço carrega o **estado atual** do agregado e valida a **transição** (o estado atual permite
   este comando?).
3. Se o modelo do comando declara **regra de negócio** ou **coordenação**, o serviço chama o
   **br-service** para validar/enriquecer/coordenar **antes** de concluir.
4. O serviço **grava o evento** no banco de escrita e responde ao cliente com o identificador do
   agregado e seu estado resultante.

A resposta é síncrona; o que acontece depois (projeção) é assíncrono.

---

## Fase 2 — Notificação e despacho (assíncrona)

5. A gravação do evento, na mesma transação, **notifica** o **es-n**.
6. O es-n lê o(s) evento(s) com garantia de **processamento exatamente-uma-vez** (avanço de checkpoint),
   reconstitui o **estado atual do agregado em JSON** e o **despacha** para filas do **broker de
   mensagens**. Há até **três** destinos:
   - **projeção** (caminho comum) — atualizar a projeção do próprio contexto;
   - **projeção cross-contexto** — atualizar projeção em outro contexto/tenant;
   - **coordenação** — disparar uma saga entre agregados/contextos.

> **Ponto de coordenação:** o que distingue os três destinos é declarado no **modelo do evento**
> (marcadores de projeção e de coordenação). Ver [coordenação](coordenacao.md).

---

## Fase 3 — Projeção

7. O consumidor da **fila de projeção** aplica o evento à **projeção** no banco de leitura
   (cria/atualiza/remove a linha correspondente), com **deduplicação** por evento para idempotência e
   **ordenação** configurável (nenhuma / estrita / por-agregado).

Ao fim desta fase, o banco de leitura reflete o estado atual do agregado.

---

## Fase 4 — Consulta

8. O cliente consulta o **persistence-q** com um critério; o serviço lê a **projeção materializada** e
   devolve os resultados. Não toca no banco de escrita nem reprocessa eventos.

---

## Coordenação cross-agregado / cross-contexto

Quando um evento exige articular outros agregados/contextos, o es-n o encaminha pela **fila de
coordenação**. O consumidor dessa fila aciona o **br-service** (função de coordenação) e, conforme o
resultado, **um novo comando** é submetido ao persistence-crs — fechando o ciclo de uma **saga**.

O inventário detalhado desses pontos de articulação está em [coordenação](coordenacao.md).

---

<a id="origem-das-filas"></a>
## De onde vêm as filas (origem da topologia de mensageria)

Há **duas** origens distintas de filas na plataforma — não confundir:

| Origem | O que cria | Como | Dinâmico? |
|---|---|---|---|
| **Filas internas de eventos** (es-n) | filas de **projeção** e os canais **universais** de projeção cross-contexto e de coordenação — **todas globais/fixas**, um conjunto compartilhado por **todos** os tenants (nomes fixos, sem fila por tenant) | declaradas pelo es-n na **inicialização**, uma única vez | **não** — topologia **fixa e global**; o tenant/contexto de destino viaja no **conteúdo** da mensagem, não em fila dedicada |
| **Filas de processo** (forger) | exchange + fila + fila de descarte + chave de roteamento de um **processo** | provisionadas no **deploy de `process` (BPMN)**, a partir dos elementos de extensão do diagrama | sim, por deploy — UPSERT por fila, inativação de órfãos, compensação em falha |

Consequências para quem constrói sistemas:

- **O deploy de `*.model.json` NÃO cria filas.** Os marcadores de evento `triggerProjection` /
  `triggerCoordination` apenas fazem o agregado ser **despachado** pelos canais universais já
  existentes do es-n; o **destino** (tenant/contexto/projeção/coordenação) viaja no **conteúdo** da
  mensagem, não numa fila dedicada por par de contextos.
- **Filas sob demanda** na plataforma vêm **somente** do deploy de **process (BPMN)** no forger.
- es-n **não** cria filas dinamicamente em runtime; só **publica** nos canais que declarou no startup.

Detalhe por serviço: [es-n](es-n/README.md) e [forger/process](forger/endpoints/process.md).

---

## Fatos transversais — fonte canônica

Alguns fatos repetem-se em vários documentos (para leitura local completa). Para evitar divergência,
a **fonte da verdade** de cada um é:

| Fato transversal | Documento canônico |
|---|---|
| Projeção é **assíncrona** (não há leitura imediata após o comando) | [persistence-q — consistência eventual](persistence-q/README.md#ciclo-de-vida-de-uma-consulta) |
| **Origem das filas** (es-n fixo × forger/process sob demanda) | [#origem-das-filas](#origem-das-filas) (acima) |
| Gramática do **`.model.json`** (agregado/comando/evento/tipos) | [model-format.md](persistence-crs/spec/model-format.md) |
| Pontos de **coordenação** entre serviços (CP-n) | [coordenacao.md](coordenacao.md) |
| **Ordem de deploy** dos recursos | [03-fluxo-de-deploy.md](03-fluxo-de-deploy.md) |
| Vocabulário **abstrato de infraestrutura** | [02-conceitos.md](02-conceitos.md#vocabulário-de-infraestrutura-termos-abstratos) |

Ao corrigir um desses fatos, **edite primeiro o documento canônico** e propague.
