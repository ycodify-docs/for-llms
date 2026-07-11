# Coordenação entre serviços

> Inventário dos pontos de articulação entre os oito serviços: onde um serviço **depende de** ou
> **provoca** outro. Consolidado a partir do contrato de cada serviço, para que um agente entenda a
> arquitetura sem ler o código.
> Pré-requisitos: [conceitos](02-conceitos.md), [arquitetura](01-arquitetura.md).

## Contents
- Como ler esta tabela
- CP-1 … CP-12 (pontos de coordenação)

## Como ler esta tabela

Cada ponto descreve: **origem → destino**, o **gatilho** (o que dispara), o **meio** (síncrono HTTP,
cache, ou fila assíncrona) e a **dependência** (o que precisa existir antes).

---

## Pontos de coordenação

### CP-1 — forger publica o modelo de domínio; os demais leem
- **Origem → destino:** forger → persistence-crs, es-n, persistence-q
- **Gatilho:** publicação do **model** (passo 5 do [deploy](03-fluxo-de-deploy.md)).
- **Meio:** **cache distribuído** (o modelo fica publicado; os serviços leem sob demanda).
- **Dependência:** o `tenant-id` (dataschema) deve existir; o modelo referencia agregados/comandos/eventos.
- **Por quê:** o modelo é a **fonte da verdade** sobre quais agregados, comandos, eventos, transições
  e projeções existem. Sem ele publicado, os serviços não têm o que interpretar.

### CP-2 — forger cria as projeções que persistence-q consulta
- **Origem → destino:** forger → (banco de leitura) → persistence-q
- **Gatilho:** criação de **entity** (passo 4 do deploy).
- **Meio:** **banco de leitura** (tabela materializada).
- **Dependência:** dataschema existente.
- **Por quê:** persistence-q só consegue consultar projeções cujas tabelas foram criadas por forger.
- **Derivação:** a projeção (entity) **deriva do agregado** (colunas ← `data.attribute`) e é criada sob
  o dataschema cujo nome = o do bounded context. Ver
  [conceitos — do agregado à projeção](02-conceitos.md#do-agregado-à-projeção-derivação-e-implantação).

### CP-3 — persistence-crs grava evento e notifica es-n
- **Origem → destino:** persistence-crs → es-n
- **Gatilho:** gravação de um **evento** (comando bem-sucedido).
- **Meio:** **notificação do banco de escrita**, na mesma transação da gravação.
- **Dependência:** modelo publicado (CP-1).
- **Por quê:** desacopla a escrita (síncrona) da atualização de projeção (assíncrona).

### CP-4 — es-n despacha o agregado para até três destinos
- **Origem → destino:** es-n → persistence-crs (projeção) · es-n → projeção cross-contexto · es-n → coordenação
- **Gatilho:** evento notificado (CP-3).
- **Meio:** **filas** do broker de mensagens (fila de projeção, fila de projeção cross-contexto, fila de coordenação).
- **Dependência:** o **modelo do evento** declara os marcadores que definem cada destino.
- **Por quê:** um único evento pode atualizar a projeção do próprio contexto, projeções de outros
  contextos e/ou disparar uma saga.
- **Topologia:** os canais de projeção cross-contexto e de coordenação são **universais e fixos**
  (declarados na inicialização do es-n); o destino viaja no **conteúdo** da mensagem, não numa fila
  dedicada por par de contextos. Os marcadores do evento **não criam filas**. Ver
  [arquitetura — de onde vêm as filas](01-arquitetura.md#origem-das-filas).
- **Critérios de roteamento** (chave `event.<endState>.domainBus` do modelo):

  | Marcador em `domainBus` | Destino | Fila/exchange |
  |---|---|---|
  | (**sempre**, mesmo sem marcador) — `orderingMode`: `per-aggregate` (default, partição por `hash(aggregateid)`) · `strict` (ordem global, consumidor único) · `none` | projeção do **próprio** contexto (banco de leitura) | fila de projeção global |
  | `triggerProjection[]` (itens **objeto** `{name,targetTenantId,targetSchema,br?}`) | projeção **cross-contexto** | exchange de projeção cross-contexto |
  | `triggerCoordination[]` (`{name,targetTenantId,br}`) | **coordenação (saga)** | exchange de coordenação |

  > `triggerDomain` **NÃO é consumido** pelo es-n (aspiracional — só parseado por ferramentas de geração de código). Coordenação **viva** = `triggerCoordination`.

### CP-5 — consumidor da fila de projeção atualiza o banco de leitura
- **Origem → destino:** (fila de projeção) → persistence-crs → banco de leitura
- **Gatilho:** mensagem na fila de projeção (CP-4).
- **Meio:** **fila assíncrona**; aplicação idempotente com deduplicação por evento e ordenação configurável.
- **Por quê:** materializa o estado atual do agregado para leitura por persistence-q.

### CP-6 — persistence-crs chama br-service (regra/coordenação)
- **Origem → destino:** persistence-crs → br-service
- **Gatilho:** o **modelo do comando** (ou do evento de coordenação) declara uma rota de regra de
  negócio ou de coordenação.
- **Meio:** **HTTP síncrono**; br-service localizado via **registro de serviços**.
- **Dependência:** o processador (função) referenciado deve existir no br-service.
- **Restrição de fronteira:** br-service é **sempre** provocado pelo persistence-crs, nunca diretamente
  pelo cliente. Seus processadores devem ser **idempotentes**; alguns (coordenação/projeção) fazem
  **leituras** de enriquecimento, mas não devem ter efeitos de escrita externos no caminho crítico.
- **Por quê:** separa a lógica de regra/coordenação (em funções) do motor de comandos.

#### Três contextos de invocação do br
O br-service é acionado em três situações distintas; só a última é obrigatória:

| Contexto | Quando | Síncrono? | Obrigatório? |
|---|---|---|---|
| **Validação/enriquecimento** | durante a execução do comando, se o modelo do comando declara uma rota de regra (`command.br.route`). | sim | não (só se declarado) |
| **Transformação cross-contexto** | ao aplicar uma projeção em outro contexto (despacho `triggerProjection` do es-n). | assíncrono (via fila) | não (só se declarado) |
| **Coordenação (saga)** | ao processar um evento de coordenação (`triggerCoordination`). | assíncrono (via fila) | **sim** — coordenação sem função de br é erro de modelo. |

Os três acima postam em **`POST /br`** (a rota distingue o papel pelo sufixo: `…/<comando>`, `…/projection/<n>`, `…/coordination/<n>` — ver [br-service](br-service/README.md)).

> **⚠️ Quarto ponto — coordenação command-path (síncrona) — GAP de plataforma.** Além dos três acima, o motor de comandos aciona uma coordenação **síncrona no caminho do comando** quando o **modelo do comando** declara `command.<c>.coordination.route`: o persistence-crs posta em **`POST /coordination`** (não `/br`), corpo `{content:{route,data}}` (agregado primário + os `coordination.references`), esperando uma **lista ordenada de comandos** executada na mesma transação. **Hoje o br-service NÃO implementa `/coordination`** (só `/br`) — mecanismo **pendente de handler** (gap de plataforma). Para coordenação, prefira `event.domainBus.triggerCoordination` (assíncrono, `/br`).

### CP-7 — coordenação fecha o ciclo da saga
- **Origem → destino:** (fila de coordenação) → persistence-crs → br-service → (novo) comando
- **Gatilho:** mensagem na fila de coordenação (CP-4, destino coordenação).
- **Meio:** **fila assíncrona** → HTTP para br-service → **novo comando** ao persistence-crs.
- **Por quê:** permite que um evento num agregado desencadeie comandos em outros — o mecanismo de saga.

#### Uso ideal dos hooks (advisory)
Os pontos de br-service são **hooks de código** (funções **JavaScript** injetadas no br-service). O *como* usar vai da **conveniência do desenvolvedor/arquiteto** — o padrão recomendado:

- **Hook de regra de negócio** (`command.br.route`, síncrono, pré-persist): a **regra de negócio propriamente dita** — validar, transformar e **enriquecer** os dados do comando; pode **abortar** (≠200). Ponto ideal para invariantes e políticas.
- **Hooks de coordenação** (`triggerCoordination` assíncrono; command-path síncrono quando houver handler): oportunidade para **efeitos e integração** — **notificar**, **enriquecer** dados que seguem no fluxo, **chamar APIs externas**, aplicar **anticorrupção** (traduzir modelo de terceiro), e **disparar o próximo comando** (saga). Recebe o **estado do agregado de origem** (`aggregateState`) para decidir.
- **Hook de projeção cross-contexto** (`triggerProjection`, assíncrono): **transformar** o evento na linha da projeção de outro contexto/tenant.

> **Fronteira (regra dura):** o br-service é sempre acionado **pelo persistence-crs** (nunca pelo cliente). Processadores devem ser **idempotentes**; efeitos colaterais (notificação/API externa) devem **tolerar re-execução** (entrega ao-menos-uma-vez). Sem escrita externa no caminho crítico do comando.
>
> **Callback (ler/escrever de volta):** um processor lê projeção (persistence-q) ou estado/escreve comando (persistence-crs) via **endpoint interno de cluster** (intra-cluster, `X-Tenant-Id`, **sem JWT** no hook async; no síncrono há `body.authToken`). ⛔ nunca hardcodar JWT nem o path interno (vem de env no deploy). Regra + comportamento: [br-service — Callback](br-service/README.md#callback-do-processor--persistence-q--persistence-crs-endpoint-interno-de-cluster).

### CP-8 — orgid valida unicidade da organização junto ao forger
- **Origem → destino:** orgid → forger
- **Gatilho:** criação de **organização** (orgid).
- **Meio:** **HTTP síncrono** (consulta interna).
- **Por quê:** garante que a org não tenha recursos preexistentes antes de criá-la.

### CP-9 — orgid persiste/consulta contratos via persistência
- **Origem → destino:** orgid → persistence-crs (escrita) / persistence-q (leitura)
- **Gatilho:** criar/atualizar/ler **contrato** de uma organização.
- **Meio:** **HTTP síncrono** aos serviços de persistência.
- **Por quê:** o contrato (assinatura/plano) é mantido como dado da plataforma.

### CP-10 — orgid fornece organização e papéis que governam a autorização
- **Origem → destino:** orgid → forger, persistence-crs, … (todos os que autorizam)
- **Gatilho:** qualquer operação autorizada (ex.: criar recurso no forger).
- **Meio:** o **token** carrega organização + papéis (originados no orgid); os serviços conferem o
  **papel na organização**.
- **Dependência:** a **organização** (`{org}`) e a **conta com papel** devem existir no orgid **antes**.
- **Por quê:** orgid é o **pré-requisito de identidade** — define **quem** é e **o que pode**, escopado
  por organização. É a base do controle de acesso de toda a plataforma.

### CP-12 — auth emite o token que todos os demais validam
- **Origem → destino:** cliente → auth (emite) → todos os serviços (validam)
- **Gatilho:** login (`/up`/`/ua` `sign-in`) ou renovação.
- **Meio:** **HTTP**; auth → orgid (validar credenciais, obter usuário/papéis/orgs); auth → forger
  (no `/ua`, descobrir tenants); auth → cache (registrar sessão, sessão única).
- **Dependência:** identidade existente no orgid (org/conta/papéis); o token embute org/papéis/tenants.
- **Por quê:** **auth emite** o token de acesso; **forger, persistence-crs, persistence-q, br-service,
  filer, orgid** o **validam** (segredo compartilhado) para autorizar. É o **início da cadeia de
  autorização**: autentique no auth → use o token nos demais.

### CP-11 — filer anexa arquivos a agregados
- **Origem → destino:** cliente/serviços → filer (e filer → modelo do tenant, para autorizar)
- **Gatilho:** upload/download/list/delete de arquivo.
- **Meio:** **HTTP** com `Authorization` + `X-Tenant-Id`; chave do arquivo
  `{org}-{project}-{entity}-{entityId}-{attribute}`.
- **Dependência:** o arquivo é **anexo de um agregado** — `entity` = tipo do agregado (forger entity),
  `entityId` = `aggregateid` (persistence-crs), `attribute` = atributo. A **autorização** usa o
  **controle de acesso do agregado** no modelo do tenant (mesmo modelo publicado pelo forger, CP-1).
- **Por quê:** desacopla o **conteúdo binário** (arquivos) do estado do agregado, mantendo-o
  **endereçável pela mesma taxonomia** de org/projeto/agregado/atributo. Um agregado pode ter vários
  arquivos, convencionalmente classificados por atributo.

---

> Este inventário é atualizado conforme cada serviço é documentado em detalhe. Cada `CP-n` é referenciado
> pelos `README.md` dos serviços envolvidos.
