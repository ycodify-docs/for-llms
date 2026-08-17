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
- Três categorias de processador (inclui **Callback do processador** — qual endpoint usar e como montar a URL)
- Pontos de coordenação
- Pitfalls

---

## Posição no fluxo

Durante a execução de um comando, se o **model** do comando (ou de um evento de coordenação) declara
uma **rota** de regra/coordenação, o persistence-crs chama o br-service nessa rota, passando os dados.
O br-service executa a **função** correspondente e devolve o resultado, que o persistence-crs incorpora
ao processamento (CP-6/CP-7). O br-service **não é chamado diretamente pelo cliente** (só pelo
persistence-crs). Um processador **pode ler de volta** persistence-q/crs para enriquecimento — ver
[Callback do processador](#callback-do-processador), que define **qual** endpoint é utilizável em cada
tipo de hook — e, quando o caso de uso exigir, chamar integrações externas nos hooks de coordenação
(assíncronos); no caminho **crítico do comando** (síncrono), evite escrita/efeitos externos.

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

> **O serviço NÃO valida a forma da rota.** Ele apenas casa a `route` recebida com o caminho do
> processador publicado — qualquer caminho funciona. A unicidade global descrita acima é garantida
> **pela disciplina de quem publica**, não por checagem em tempo de execução. Consequência concreta de
> abrir mão do prefixo: duas organizações que publiquem `pedido/criar` acabam apontando para o **mesmo
> caminho** e, portanto, para o **mesmo processador** — a regra de uma passa a valer, silenciosamente,
> para a outra.

### Deploy de um processador (por sistema de arquivos — NÃO via gateway)

Um processador é um **arquivo `.js`** cujo **caminho na pasta de processadores do serviço** é a **rota**
(`<org>/<project>/<bc>/<aggregate>/<função>.js`). **Não há endpoint/API para publicar um processador** — o
br-service é **interno** (sem rota no gateway). O deploy é por **sistema de arquivos**: coloca-se o `.js` na pasta
de processadores e o serviço o carrega (na inicialização; recarregar torna a rota viva). Uma rota sem processador
correspondente → **erro de rota** quando o persistence-crs a aciona.

## Contrato de resposta

- Sucesso (`200`): o serviço devolve **diretamente** o objeto retornado pelo processador (sem envelope).
  Esse objeto é o que o persistence-crs usa para enriquecer/validar/coordenar o comando.
- Falha (`400`): **duas formas distintas**, conforme o ponto em que falhou:

| Falha | Corpo da resposta |
|---|---|
| **Validação do corpo** (corpo vazio/não-objeto/sem `route`) | `{ "erro": "<mensagem>" }` |
| **Rota inexistente** ou **exceção no processador** | `{ "status": "error", "mensagem": "<mensagem>", "tipo": "<tipo do erro>" }` |

## Ciclo de vida de uma requisição

`POST /br`:

1. **Validação** — corpo não-nulo, objeto, com `route`. Falha → `400` (forma `{erro}`).
2. **Roteamento** — localiza o processador pela `route`. Não encontrado → `400` (a mensagem lista as
   rotas disponíveis).
3. **Execução** — o processador recebe **de um a três argumentos**, conforme o que veio no corpo:

   | Corpo recebido | Argumentos entregues ao processador |
   |---|---|
   | `data` **+** `authToken` **+** `tenantIds` | `(data, authToken, tenantIds)` |
   | `data` **+** `authToken` | `(data, authToken)` |
   | `data` | `(data)` |
   | **sem** `data` | `(corpo inteiro)` |

   `authToken` e `tenantIds` só aparecem na raiz do corpo no hook **síncrono** (regra de negócio) — é o
   motor de comandos que os coloca lá. `tenantIds` é um **array** com o `tenantId.forReadModel` declarado
   no modelo do agregado (ver [model-format](../persistence-crs/spec/model-format.md)); um `authToken`
   nulo (comando sem credencial) não conta — nesse caso o processador cai no modo de **um argumento**
   (`data` apenas), e `tenantIds` **não é entregue** mesmo que presente no corpo.

   O processador pode ser assíncrono; o resultado é aguardado.
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

### Forma do `data` por tipo de hook

O campo `data` entregue ao processador tem **forma diferente** em cada categoria. Misturar as formas
causa `undefined` silencioso (ex.: `data._meta.targetTenantId` num hook de coordenação retorna `undefined`).

**1) Regra de negócio — síncrono (`command.br.route`)**

`data` = payload bruto do comando (campos de `command.<cmd>.data.attribute[]` do `.model.json`).
`authToken` e `tenantIds` chegam como **argumentos separados** — não dentro de `data`.

**2) Coordenação saga — assíncrono (`triggerCoordination`)**

```js
data = {
  "<entityName>": { ...eventData },   // dados do evento de origem
  aggregateState: { ...fullState },   // estado completo do agregado de origem (carregado pelo crs)
  targetTenantId: "<uuid>",           // tenant destino — top-level em data (NÃO em _meta)
  sourceTenantId: "<uuid>"            // tenant origem  — top-level em data (NÃO em _meta)
}
// authToken chega na raiz do corpo → entregue como segundo argumento se não nulo
```

`aggregateState` é a **única** forma de ler o estado atual do agregado de origem dentro do processador.

**3) Projeção cross-contexto — assíncrono (`triggerProjection`)**

```js
data = {
  "<entityName>": { ...eventData },   // dados do evento de origem
  _meta: {
    targetTenantId: "<uuid>",         // tenant destino — dentro de _meta (≠ coordenação)
    sourceTenantId: "<uuid>",         // tenant origem  — dentro de _meta
    authToken?: "<jwt>"               // JWT de origem quando propagado; pode estar ausente
  }
}
// authToken NÃO chega como argumento — leia de data._meta.authToken (oportunista)
```

⚠️ **Assimetria crítica:** em coordenação `targetTenantId`/`sourceTenantId` são **top-level** em `data`;
em projeção cross-contexto ficam em **`data._meta`**. `aggregateState` **só existe** no hook de coordenação.

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

<a id="callback-do-processador"></a>
### Callback do processador → persistence-q / persistence-crs

O br-service roda **intra-cluster** (atrás da DMZ). Quando um processador precisa **ler** uma projeção
(enriquecimento), **ler** o estado de um agregado ou **escrever** um comando de volta, ele chama a
persistence-q/crs diretamente.

Ponto de partida: **só o hook síncrono (regra de negócio) recebe JWT de forma garantida** — ele chega em
`authToken` na raiz do corpo e é entregue ao processador como **argumento**.

Nos hooks **assíncronos** o comportamento do JWT difere por categoria:

- **Coordenação (`triggerCoordination`):** o token do usuário de origem é propagado na **raiz do corpo**
  (`authToken`) → o processador o recebe como **segundo argumento**, igual ao hook síncrono. Trate-o como
  oportunista: quando o evento de origem não carregou JWT, o campo virá nulo e o processador cairá no modo
  de um argumento.
- **Projeção cross-contexto (`triggerProjection`):** o token — quando propagado — chega **dentro** de
  `data._meta.authToken` (não como argumento separado). Nem toda projeção recebe token. Um processador de
  projeção **não pode depender** de ter JWT: sempre tenha o caminho sem token.

Cada operação existe em **duas superfícies espelhadas**, com as mesmas rotas e o mesmo corpo:

| Superfície | Path | Headers | Utilizável em |
|---|---|---|---|
| **Pública** (via gateway) | o path documentado do serviço | `Authorization` **+** `X-Tenant-Id` | hook **síncrono**; num assíncrono, **só** se o token tiver vindo em `_meta.authToken` |
| **Interna de cluster** | **prefixo próprio** + o mesmo path | **só** `X-Tenant-Id` — **sem** `Authorization` | qualquer hook — é a única opção que **sempre** funciona no assíncrono |

O tenant sai do próprio payload (`targetTenantId`/`sourceTenantId`).

A superfície interna não tem controle de acesso por conta de usuário: é feita para **chamadas internas**,
o gateway **nega** esses paths, e ela só é alcançável de dentro do cluster. O prefixo é **sensível** e por
isso **não consta desta documentação** — a plataforma o injeta no ambiente do serviço (ver abaixo).

> **Erro comum:** supor que basta apontar para o endereço-base do serviço. **Não basta.** O endereço-base
> leva à superfície **pública**, que exige `Authorization` — e um hook assíncrono pode muito bem não ter
> token nenhum. A superfície interna fica sob um **prefixo próprio**, obtido conforme a seção seguinte.

| Ação | Serviço · path (o mesmo nas duas superfícies) | Corpo |
|---|---|---|
| **Ler projeção** | persistence-q · endpoint de execução de consulta | `{"<projeção>":{<predicado>}}` → `200` array · `204` vazio |
| **Ler estado do agregado** | persistence-crs · `…/a/<bc>/<aggType>/<uuid>` [+ `/history`] | — |
| **Escrever comando** | persistence-crs · `…/a/<bc>/<aggType>` | `{"<comando>":{...}}` |

### Como o processador obtém a URL (variáveis de ambiente)

Endereço e path são injetados pela plataforma no **ambiente do serviço** e ficam disponíveis a
**qualquer** processador, em qualquer nível de pasta. Nada disso vem no payload:

| Variável de ambiente | Conteúdo |
|---|---|
| `PLATFORM_ENDPOINT_PERSISTENCE_C` | endereço-base do persistence-crs (terminado em `/`) |
| `PLATFORM_ENDPOINT_PERSISTENCE_Q` | endereço-base do persistence-q (terminado em `/`) |
| `PLATFORM_ENDPOINT_PERSISTENCE_Q_UNSEC_PATH` | **path da superfície interna** do persistence-q — o endpoint de execução de consulta, sem `Authorization` |
| `PLATFORM_ENDPOINT_ORGID` | endereço-base do orgid (para verificar/criar conta de usuário de app via `ENDPOINTS.ORGID`) |

- Montagem da URL interna de leitura de projeção: `PLATFORM_ENDPOINT_PERSISTENCE_Q` +
  `PLATFORM_ENDPOINT_PERSISTENCE_Q_UNSEC_PATH`, com exatamente uma `/` entre os dois.
- Acesso ao orgid: `ENDPOINTS.ORGID` → `GET .../open/ua/account/by/username/{username}/exists` (`200` = conta existe, `204` = não existe). Ver `orgid/endpoints/publico.md`.
- **Helper recomendado — `lib/platform-endpoints.js`:** em vez de ler `process.env.PLATFORM_ENDPOINT_*`
  diretamente, use o módulo `lib/platform-endpoints` (disponível em qualquer processador, independente de
  profundidade de pasta):
  ```js
  const ENDPOINTS = require('../../../lib/platform-endpoints'); // ajuste ../ à profundidade
  // ENDPOINTS.PERSISTENCE_C, ENDPOINTS.PERSISTENCE_Q, ENDPOINTS.PERSISTENCE_Q_UNSEC, ENDPOINTS.ORGID
  ```
  O módulo usa **getters lazy** — a variável ausente lança erro nomeado na requisição (não no `require`),
  mantendo o processador no mapa de rotas mesmo com env incompleto.
- Variável **ausente** → o processador falha com `400`, e a mensagem **nomeia a variável faltante**.
- Para o persistence-crs **não há**, hoje, variável com o prefixo da superfície interna. Um hook
  assíncrono que precise escrever comando usa a superfície **pública** com um **token de serviço vindo do
  ambiente** — nunca um token escrito no processador.
- ⛔ **Nunca hardcodar** endereço, prefixo interno nem token no processador: todos mudam por ambiente, e o
  valor gravado no arquivo fica errado no deploy seguinte.

> ⛔ **Nunca hardcodar um JWT** no processador. O único JWT legítimo é o `authToken` que chega no corpo
> do hook **síncrono**. Hook assíncrono não tem JWT — e a resposta para isso é o endpoint interno acima,
> **nunca** um token gravado no arquivo.

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
- [ ] Escolher a superfície pelo **tipo de hook**: assíncrono (coordenação/projeção) → superfície **interna** (`X-Tenant-Id`, sem `Authorization`); síncrono → pode usar a **pública** com o `authToken` do corpo.
- [ ] **Não** apontar para o endereço-base puro num hook assíncrono — isso cai na superfície **pública**, que exige `Authorization`.
- [ ] Endereço e path saem de `PLATFORM_ENDPOINT_PERSISTENCE_C` / `PLATFORM_ENDPOINT_PERSISTENCE_Q` / `PLATFORM_ENDPOINT_PERSISTENCE_Q_UNSEC_PATH`. ⛔ **Nunca** gravar destino de rede, path interno ou JWT no processador.
- [ ] Usar `lib/platform-endpoints` (`ENDPOINTS.*`) em vez de ler `process.env` diretamente — getters lazy nomeiam a variável faltante no erro.
- [ ] Em hook síncrono com `authToken` nulo: o processador cai em modo de **um argumento** — `tenantIds` **não** é entregue. Não assuma que `tenantIds` chegou se o comando não carregou JWT.
- [ ] Em hook de coordenação: `targetTenantId`/`sourceTenantId` são **top-level** em `data` (não em `_meta`). Em projeção cross-contexto: ficam em `data._meta`. Não trocar os dois.
