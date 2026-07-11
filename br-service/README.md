# br-service — guia do serviço

> **Papel:** executa **regras de negócio** e funções de **coordenação** de agregados (comumente em
> sagas). Recebe dados e devolve dados; processadores devem ser **idempotentes** e, idealmente, puros
> (alguns de coordenação/projeção fazem **leituras** — ver ressalva abaixo).
> **Quem chama:** **somente** o [persistence-crs](../persistence-crs/README.md) — nunca o cliente final
> diretamente (CP-6). Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Contents
- Posição no fluxo
- Índice de endpoints
- Roteamento de processadores
- Contrato de resposta
- Ciclo de vida de uma requisição
- Autoria de processadores
- Pontos de coordenação
- Pitfalls

---

## Posição no fluxo

Durante a execução de um comando, se o **model** do comando (ou de um evento de coordenação) declara
uma **rota** de regra/coordenação, o persistence-crs chama o br-service nessa rota, passando os dados.
O br-service executa a **função** correspondente e devolve o resultado, que o persistence-crs incorpora
ao processamento (CP-6/CP-7). O br-service **não é chamado diretamente pelo cliente** (só pelo
persistence-crs). Um processor **pode ler de volta** persistence-q/crs para enriquecimento (via **endpoints
internos de cluster**, ver abaixo) e — quando o caso de uso exigir — chamar integrações externas nos hooks de
coordenação (async); no caminho **crítico do comando** (síncrono), evite escrita/efeitos externos.

## Índice de endpoints

> Apenas o endpoint de contrato é documentado. Endpoints de saúde/observabilidade **não** constam aqui.

| Operação | Método · Path | Documento |
|---|---|---|
| Executar função (rota) | `POST /br` | [endpoints/br.md](endpoints/br.md) |

> **⚠️ `POST /coordination` — pendente (gap de plataforma).** O motor de comandos (persistence-crs)
> aciona uma coordenação **síncrona no caminho do comando** quando o modelo do comando declara
> `command.<c>.coordination.route`, postando em **`POST /coordination`** (corpo `{content:{route,data}}`,
> retorno = lista ordenada de comandos). **O br-service ainda NÃO implementa este endpoint** (só `/br`).
> Até implementá-lo, a coordenação command-path não funciona — use a **coordenação assíncrona**
> `event.domainBus.triggerCoordination[]` (posta em `/br`, rota `…/coordination/<n>`; ver "Três categorias").

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md).

## Roteamento de processadores

- Cada **função** é um **processador** identificado por uma **rota** (caminho textual).
- O persistence-crs envia `{ "route": "<rota>", "data": { ... } }`; o serviço localiza o processador
  pela rota e o executa com os dados.
- Rota inexistente → erro (com a lista de rotas disponíveis).

### Forma canônica da rota (obrigatória)

O br-service serve **múltiplas organizações**, cada uma com uma hierarquia de recursos semelhante. Para
**evitar colisão** entre funções de organizações diferentes, a rota é **totalmente qualificada**, do
mais geral ao mais específico:

```
<org>/<project>/<boundedContext>/<aggregate>/<função>
```

- **Regra de negócio:** `<org>/<project>/<bc>/<aggregate>/<comando>`
- **Coordenação:** `<org>/<project>/<bc>/<aggregate>/coordination/<nome>`
- **Projeção:** `<org>/<project>/<bc>/<aggregate>/projection/<nome>`

**Por que isso garante unicidade global** — cada segmento é único **dentro do seu pai**:

| Segmento | Único dentro de |
|---|---|
| `org` | espaço de nomes global |
| `project` | da organização |
| `boundedContext` | do projeto |
| `aggregate` | do bounded context |
| `função` (comando / coordination/nome / projection/nome) | do agregado |

Aninhando os escopos, o **path completo é globalmente único** → duas organizações podem ter o mesmo
`pedido/criar` sem conflito, pois os prefixos `<org>/<project>/<bc>` diferem.

> Pela [convenção padrão](../02-conceitos.md#convenção-padrão-de-topologia-project--bounded-context--esquema)
> (1 project = 1 bounded context), os segmentos `project` e `boundedContext` costumam **coincidir** —
> ex.: `acme/vendas/vendas/pedido/criar`. Não é erro: são escopos distintos que, por padrão, têm o mesmo nome.

## Contrato de resposta

- Sucesso (`200`): o serviço devolve **diretamente** o objeto retornado pelo processador (sem envelope).
  Esse objeto é o que o persistence-crs usa para enriquecer/validar/coordenar o comando.
- Falha (`400`): objeto com `status`, mensagem e tipo do erro.

## Ciclo de vida de uma requisição

`POST /br`:

1. **Validação** — corpo não-nulo, objeto, com `route`. Falha → `400`.
2. **Roteamento** — localiza o processador pela `route`. Não encontrado → `400` (lista rotas disponíveis).
3. **Execução** — invoca o processador com `data` (ou o corpo inteiro, se `data` ausente). Pode ser
   assíncrono; o resultado é aguardado.
4. **Resposta** — `200` com o resultado **direto** do processador; exceção no processador → `400` com
   `{ status, mensagem, tipo }`.

## Três categorias de processador

A **rota** de um processador sinaliza seu papel, por convenção de segmento de path. Os três casos
correspondem aos três pontos onde o modelo referencia uma rota de br:

> Todas as rotas seguem a [forma canônica](#forma-canônica-da-rota-obrigatória)
> `<org>/<project>/<bc>/<aggregate>/…`. As colunas abaixo mostram só o **sufixo** que distingue a categoria.

| Categoria | Referenciado em | Sufixo da rota | Entrada | Retorno (em `processedData`) |
|---|---|---|---|---|
| **Regra de negócio** | `command.br.route` | `…/<aggregate>/<comando>` (nome do comando) | dados do comando | os **dados validados/enriquecidos** do comando — mesclados no comando antes de gravar o evento |
| **Coordenação (saga)** | `event.domainBus.triggerCoordination[].br.route` | `…/<aggregate>/coordination/<alvo_from_origem>` (segmento **`coordination`**) | dados do evento (estado do agregado de origem) | um **`targetCommand`** `{ boundedContext, aggregateType, commandName, data }` — submetido como novo comando no contexto destino (CP-7) |
| **Projeção cross-contexto** | `event.domainBus.triggerProjection[].br.route` | `…/<aggregate>/projection/<alvo_from_origem>` (segmento **`projection`**) | dados do evento (estado do agregado de origem) | a **linha da projeção destino** `{ "<projeção>": { …campos } }` — aplicada na projeção do outro contexto |

**Convenção de nome** (coordenação/projeção): `<alvo>_from_<origem>` (ex.: `conta_from_pedido`).

> **⚠️ Regra de negócio — o merge de volta é whitelist (chaves novas são descartadas):** os dados
> retornados pelo processor são mesclados no comando **apenas nas chaves que já existem** no
> `commandData`; **chaves NOVAS retornadas pelo processor são DESCARTADAS** (não chegam ao evento
> gravado). Por isso todo campo que o processor pretende **enriquecer/calcular** já precisa existir como
> **atributo do comando no `.model.json`** — do contrário o valor devolvido se perde silenciosamente.

### Pureza e efeitos colaterais (importante)

- O **núcleo** do br-service é um **dispatcher**: não faz chamadas externas.
- Um **processador** é uma **função** que **deve** ser **idempotente** e, idealmente, **pura**.
- **Ressalva honesta:** processadores de **coordenação** e **projeção** **podem** fazer **leituras**
  (ex.: consultar projeções) para enriquecer o mapeamento. Isso é aceitável desde que seja
  **somente leitura**, idempotente e tolerante a reentrega. **Evite** efeitos colaterais externos com
  escrita no caminho crítico do comando.
- A rota usada é a mesma referenciada no **model** do comando/evento (ver
  [model-format](../persistence-crs/spec/model-format.md#eventos-e-domainbus)).

### Callback do processor → persistence-q / persistence-crs (endpoint interno de cluster)

O br-service roda **intra-cluster** (atrás da DMZ). Quando um processor precisa **ler** uma projeção
(enriquecimento) ou **escrever** de volta, usa os **endpoints internos de cluster** da persistence-q/crs
— sem controle de acesso por conta de usuário, feitos para **chamadas internas** (**nunca expostos fora
do cluster**). O **path exato é sensível e injetado pela plataforma no deploy** → o processor o lê de
**variável de ambiente**; **NUNCA hardcodar** o path. Header obrigatório: **`X-Tenant-Id`** (o tenant vem
do payload: `targetTenantId`/`sourceTenantId`). **Sem `Authorization`/JWT.**

| Ação | Endpoint interno de cluster | Corpo |
|---|---|---|
| **Ler projeção** (persistence-q) | endpoint interno de **leitura** da persistence-q | `{"<projeção>":{<predicado>}}` → `200` array · `204` vazio |
| **Ler estado do agregado** (persistence-crs) | endpoint interno de **leitura de agregado** (`…/a/<bc>/<aggType>/<uuid>` [+ `/history`]) | — |
| **Escrever comando** (persistence-crs) | endpoint interno de **comando** (`…/a/<bc>/<aggType>`) | `{"<comando>":{...}}` |

> **JWT — regra síncrono vs assíncrono:** no hook **síncrono** (regra de negócio `/br`, caminho do
> comando) o corpo carrega `authToken` (JWT do usuário) — o processor **pode** usar endpoints seguros
> com esse token. Nos hooks **assíncronos** (coordenação/projeção, via fila) **NÃO há JWT** → **usar o
> endpoint interno de cluster** (path via env, `X-Tenant-Id`, sem `Authorization`). ⛔ **NUNCA hardcodar
> um JWT nem o path do endpoint interno** no processor — ambos vêm do ambiente injetado no deploy.

## Pontos de coordenação
- **CP-6** — o persistence-crs chama o br-service quando o modelo do comando exige regra/coordenação.
- **CP-7** — no fluxo de saga, a coordenação devolvida pode levar o persistence-crs a submeter um novo comando.
Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Não chamar o br-service diretamente: ele é acionado **pelo persistence-crs** conforme o **model**.
- [ ] Garantir que a **rota** referenciada no model exista como processador.
- [ ] Manter o processador **idempotente**; leituras de enriquecimento são aceitáveis, mas **evite
      escrita/efeitos externos** no caminho crítico do comando.
- [ ] A resposta de sucesso é o objeto **direto** do processador (sem envelope) — modelar de acordo.
- [ ] Callback a persistence-q/crs: **hook async → endpoint interno de cluster + `X-Tenant-Id`, SEM JWT**; hook síncrono pode usar `body.authToken`. ⛔ **NUNCA hardcodar JWT nem o path do endpoint interno** (vêm de env injetada no deploy).
