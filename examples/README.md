# Exemplos — modelo de domínio

> Exemplo concreto de **modelo de domínio** (`.model.json`) e como cada parte exercita os serviços da
> plataforma, ponta a ponta. Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Arquivos

**Introdutório (curado):**
- [`acme.vendas.model.json`](acme.vendas.model.json) — um agregado (`pedido`) no contexto `vendas`, com
  estados, comandos, eventos, regra de negócio e uma saga. Genérico e completo; melhor ponto de partida.

**Reais (sanitizados):** modelos de domínio de produção, com identificadores obfuscados (UUIDs de tenant
substituídos por valores inexistentes `00000000-…-000N`, nomes de cliente/banco genericizados, referências
internas removidas). Úteis por mostrarem padrões mais ricos:
- [`acme.crm.model.json`](acme.crm.model.json) — 3 agregados (conta, oportunidade, decisão); o mais completo.
- [`acme.engenharia.model.json`](acme.engenharia.model.json) — saga cross-contexto com integração a sistema externo.
- [`acme.financeiro.model.json`](acme.financeiro.model.json) — calendário de pagamentos (1 agregado).
- [`acme.suporte.model.json`](acme.suporte.model.json) — value objects aninhados (1 agregado).
- [`gamacorp.logistica.model.json`](gamacorp.logistica.model.json) — campos semiestruturados (1 agregado).

## Anatomia do modelo

> Gramática formal e completa: [persistence-crs/spec/model-format.md](../persistence-crs/spec/model-format.md).

| Bloco | O que descreve | Quem usa |
|---|---|---|
| `type` / `org` / `project` / `boundedContext` | Tipo do agregado, organização, contexto. | forger (deploy), todos (escopo). |
| `schema` / `tenantId` | Modelo de escrita (universal) e leitura (por contexto); `tenant-id` do isolamento. | persistence-crs, es-n, persistence-q. |
| `identity` / `concurrency` | Estratégia de id (UUID gerado) e de concorrência (otimista). | persistence-crs. |
| `command` | Comandos: `data.attribute` (com `type`/`length`/`nullable`), `fromState`/`endState`, `roles`, `coordination`, `br.route` opcional. | persistence-crs (executar), br-service (regra). |
| `event` | Eventos: `type`, `whenAttribute`, `payloadInherit` e **`domainBus`** (`orderingMode`, `triggerProjection`, `triggerCoordination`). | es-n (despacho), persistence-crs (projeção/coordenação). |

Os estados **não** formam um bloco próprio: são as strings de `fromState`/`endState` dos comandos.

## Como cada parte mapeia aos endpoints

1. **Deploy (forger):** publicar este modelo com
   [`POST .../tenant/{tenantId}/model`](../forger/endpoints/model.md). A projeção do agregado (esquema
   de leitura `vendas`) deve existir como **entity** antes ([forger/entity](../forger/endpoints/entity.md)).
2. **Comando (persistence-crs):** `POST /a/vendas/pedido` com `{ "criar": { ... } }`
   ([crs/comando](../persistence-crs/endpoints/comando.md)). O serviço valida `fromState`/`endState`.
3. **Regra de negócio (br-service):** como `criar` declara `br.route`, o persistence-crs chama
   [`POST /br`](../br-service/endpoints/br.md) antes de gravar o evento (CP-6).
4. **Despacho (es-n):** ao gravar `pedidocriado`, o [es-n](../es-n/README.md) atualiza a projeção do
   contexto (caminho comum). Ao gravar `pedidofaturado`, o `domainBus.triggerCoordination` dispara a
   **saga** `abrir_cobranca` no contexto financeiro (CP-4/CP-7).
5. **Consulta (persistence-q):** ler a projeção com
   [`POST /`](../persistence-q/endpoints/consulta.md), ex.: `{ "pedidosCriados": { "status": "criada" } }`.

## Fluxo ponta a ponta (resumo)

```
forger: publica model + entities
   │
crs: POST /a/vendas/pedido { "criar": {...} }
   │   └─ br: POST /br { route:"acme/vendas/vendas/pedido/criar" }   (regra)
   │   grava evento pedidocriado ─────▶ es-n
   │                                      └─ atualiza projeção do contexto
   │
crs: POST /a/vendas/pedido { "faturar": {...} }
       grava evento pedidofaturado ───▶ es-n
                                          ├─ atualiza projeção do contexto
                                          └─ saga: abrir_cobranca (contexto financeiro)
   │
q:  POST /  { "pedidosCriados": { "status": "criada" } }
```

## Nota sobre os exemplos

Este exemplo é **genérico e autocontido**, por design: a documentação é publicada externamente e não
incorpora modelos de produção (que carregam identificadores internos). Os padrões aqui — estados,
transições, regra de negócio, projeção e saga — cobrem o que um agente precisa para modelar um domínio
real.
