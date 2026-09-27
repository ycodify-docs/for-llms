# shell · contrato casca ↔ miolo

> A **interface** entre a **casca (host)** e o **miolo (remoto)**. A casca carrega o miolo em runtime e
> o monta passando um **hostContext**; o miolo expõe um módulo conhecido — o **MioloModule**. O miolo
> **nunca** recebe token nem segredo — só o necessário para renderizar e para falar com o BFF.
> Pré: [README](README.md), [injecao](injecao.md).

## MioloModule — o que o miolo expõe

| Export | Assinatura | Papel |
|---|---|---|
| `mount` | `(el, hostContext) → dispose` | monta a UI do bounded context em `el`; devolve uma função de **desmontagem** |
| `getPresentationSchema?` | `() → PresentationSchema` | (opcional) esquema de apresentação que o miolo usa |
| `getDesignTokens?` | `() → Tokens` | (opcional) tokens de tema do miolo (ver [estilo](estilo.md)) |

O contrato **mínimo obrigatório** é `mount` (e o `dispose` que ele devolve). O resto é opcional.

## hostContext — o que a casca passa ao miolo

```jsonc
{
  "tenant":       { "org": "...", "project": "...", "boundedContext": "...", "tenantId": "..." },
  "capability":   { /* modelo de capacidade — abaixo */ },
  "aggregate":    "...",
  "presentation": { /* manifesto de apresentação — abaixo */ } | null,
  "identity":     { "username": "...", "name": "...", "email": "..." },
  "roles":        ["..."],
  "activeRole":   null,
  "api":          { "command": "…", "query": "…", "count": "…", "aggregate": "…", "history": "…" },
  "prefs":        { "formMode": "inline" | "modal" },
  "canConfigure": false,
  "savePrefs":    "(formMode) => Promise<void>",
  "openSupport":  "(contexto, focoErro?) => void",
  "diagnostico":  { "erro": "(e, extra?) => id", "trilha": "(evento, detalhes?) => void" }
}
```

- **`tenant`** — o bounded context selecionado (org/projeto/BC + `tenantId` **opaco**).
- **`aggregate`** — o agregado que o usuário escolheu **no menu da casca** (o valor de
  `capability.aggregates[].aggregate`). O menu é **Projeto → Contexto → Agregados**: o agregado se escolhe
  na casca, e cada escolha **remonta** o miolo — seleção, busca, filtros e comando em andamento recomeçam.
  O miolo mostra só esse agregado. Ausente numa casca anterior a esta regra: o miolo abre o primeiro.
- **`capability`** — o **modelo de capacidade** (abaixo): o que o usuário pode fazer ali.
- **`identity`** — dados **não-sensíveis** do usuário. **Sem token, sem senha.**
- **`roles`** — os papéis **ativos** do usuário **na organização deste tenant** (UPPERCASE, sem `ROLE_`).
  - **`activeRole`** — o papel que o usuário escolheu para esta sessão, ou `null`.
  - **Os dois são dica de apresentação, nunca autorização.** Servem para o miolo decidir o que mostrar
    — por exemplo, oferecer "reservar para outra pessoa" só a quem atende no balcão. Quem autoriza é o
    servidor, a cada chamada: o BFF e o persistence-crs revalidam o comando, e a regra de negócio confere
    o que o papel pode fazer com os dados. Esconder um campo na tela não impede ninguém de enviá-lo.
  - Não confundir com `capability.commands[].roles`, que são os papéis que o **modelo** autoriza no
    comando, e não os do usuário.
- **`api`** — as operações de domínio, todas contra o **BFF** (nunca a plataforma direto):

  | Método | Para quê |
  |---|---|
  | `command({ tenantId, aggregate, command, data, id?, status? })` | dispara um comando (escrita) |
  | `query({ tenantId, aggregate, predicates?, paging?, sorting? })` | consulta a projeção (leitura) |
  | `count({ tenantId, aggregate, predicates? })` | **quantos** registros a consulta traria sem teto, sob o mesmo filtro — para o "X de Y" |
  | `aggregate({ tenantId, aggregate, id })` | **estado autoritativo** do agregado |
  | `history({ tenantId, aggregate, id })` | histórico de eventos do agregado |

  **Comando recusado** rejeita com um erro que traz `message` (a mensagem legível da plataforma), `status`
  (o HTTP) e `em` (o instante, em epoch ms: o `timestamp` do corpo de erro, ou a hora da resposta). A
  plataforma **não** tem código de erro estável além do status nem id de correlação — o miolo não deve
  exibir nem inventar nenhum dos dois.

  O miolo **não** compõe `Authorization` nem `X-Tenant-Id` — o BFF injeta no servidor (ver
  [seguranca](seguranca.md), [bff](../bff/README.md)).

  > ⚠️ **`paging` é opcional na assinatura, não na prática.** Sem ele o read model aplica um **limite
  > padrão** (500 registros no modo array) e **não avisa**: a lista chega cortada, e a tela a exibe
  > como se estivesse completa. Um miolo que lista registros **manda `paging` sempre**.
  >
  > | campo | forma |
  > |---|---|
  > | `paging` | `{ _maxRegisters, _firstRegister }` — **os dois**, inteiros ≥ 0 |
  > | `sorting` | `{ _orderBy, _order }`, com `_order` em `ASC`/`DESC` **maiúsculo** |
  >
  > **Meio `paging` é pior que nenhum**: incompleto, a consulta roda **sem limite** e devolve a tabela
  > inteira do tenant. O BFF recusa a forma incompleta com `400` antes de ir à rede. Onde esses
  > controles ficam no critério e o que mais é recusado: [persistence-q —
  > query-controls](../persistence-q/query-controls.md).
  >
  > **Saber se há mais** sem uma segunda consulta: peça `_maxRegisters` registros; se vierem
  > exatamente esse tanto, provavelmente há mais. O total exato é o `count()`, que custa outra ida à
  > rede.

  > Use `aggregate()` — não `query()` — para carregar o estado de um agregado antes de uma transição:
  > a projeção é **assíncrona** e pode ainda não refletir o último comando. Depois de escrever,
  > **reconsulte** `query()` até o estado esperado aparecer (read-your-writes).

- **`prefs`** — preferência de apresentação **da organização** (hoje: `formMode`, o layout do form de
  comando). Vale para todos os usuários da org. Ver [bff](../bff/README.md).
- **`canConfigure`** — se **este** usuário pode alterar a preferência (é `MASTER` na org). É dica de
  UX: o BFF **revalida** no servidor.
- **`savePrefs(formMode)`** — persiste a preferência da org. Só tem efeito se o BFF confirmar o papel.
- **`presentation`** — o **manifesto de apresentação** do tenant, ou `null` quando ele não tem um. Diz
  como a tela apresenta cada agregado; **não é domínio** (o motor não o lê). É publicado no forger por
  quem tem conta de plataforma, como o `.model.json`, e o BFF o lê por tenant
  ([bff](../bff/README.md#manifesto-de-apresentação)). Chaveado pelo mesmo nome de
  `capability.aggregates[].aggregate`; tudo nele é opcional, e o que faltar o miolo deriva do nome e do tipo:

  | Chave (por agregado) | O que é |
  |---|---|
  | `singular`, `plural` | como chamar o agregado |
  | `genero` | `feminino` ou `masculino` (ausente = masculino): a tela concorda "Nova"/"Novo" |
  | `titleKey` | atributo (ou `grupo.campo`) que dá título ao registro |
  | `labels` | rótulo por atributo, por value object, ou por campo de value object como `grupo.campo` |
  | `stateLabels` | rótulo por estado |
  | `fmt` | formato por atributo ou `grupo.campo`: `date` · `datetime` · `money` · `money:<casas>` (inteiro com casas implícitas) · `phone` · `bool` · `mono` |
  | `cols`, `filters` | colunas visíveis por padrão; campos oferecidos no filtro — atributo ou `grupo.campo` (num grupo `multiple`, a coluna junta os valores dos itens e o filtro casa com qualquer item) |
  | `options` | opções de seleção, de atributo ou de `grupo.campo`: lista fixa, ou `{ aggregate, valueKey, labelKey }` de outro agregado do mesmo tenant |
  | `stateHue` | matiz (0–359) da pílula de cada estado; sem ela, a matiz sai do nome do estado |

  O nome de exibição de **comando** e de **evento** não está aqui: é o `alias` do modelo, que já vem na
  capacidade. Como preencher cada chave e publicar: [apresentacao](apresentacao.md). Forma completa e
  regras de publicação: [forger — manifesto de apresentação](../forger/endpoints/presentation.md). Referência órfã (atributo
  que o modelo não tem mais) o miolo ignora.
- **`openSupport(contexto, focoErro?)`** — abre o suporte da casca com o contexto técnico que o miolo tem
  (registro, estado, versão; e, vindo de um comando recusado, o status, a mensagem e o horário), como pares
  `{ rótulo: valor }`. A casca acrescenta tenant e tela, e o **diagnóstico da tela** (abaixo). `focoErro` é o
  id que `diagnostico.erro` devolveu: põe aquele erro à frente do relato. O relato vai à equipe da plataforma, que responde na mesma gaveta
  ([bff](../bff/README.md#relatos-de-suporte-sessionsuporte), onde está também desde quando isso vale no ar).
- **`diagnostico`** — a caixa-preta da casca, para o miolo contar o que só ele sabe. O que ela guarda vai
  junto do relato **só quando o usuário relata**, e ele vê e pode remover cada parte antes de enviar
  ([segurança](seguranca.md#diagnóstico-do-relato-de-suporte)).

  | Método | Para quê |
  |---|---|
  | `erro(e, extra?)` | registra um erro que o miolo pegou (comando recusado, falha que ele tratou) e devolve o **id** para `openSupport` |
  | `trilha(evento, detalhes?)` | anota o que o usuário fez: comando aberto, filtro aplicado, registro selecionado, aba |

  `extra` e `detalhes` são pares `{ nome: texto }` com **nomes e ids** — nome de comando, de atributo, id
  do registro. **Nunca valor de registro nem o que o usuário digitou.** A casca redige o que escapar (token,
  e-mail, CPF) e corta o que passar do teto, mas a regra é não mandar.

  Erro que o miolo **não** pega também chega ao diagnóstico: a casca escuta os erros não tratados e as
  promessas rejeitadas do documento inteiro, o que alcança qualquer miolo, GEN ou CUSTOM, e atribui ao miolo
  pelo endereço do código na pilha. Um miolo com raiz React própria deve pôr um **limite de erro** em volta
  da sua árvore: sem ele, um erro de renderização apaga a tela do miolo sem aviso. O miolo GEN já tem o seu.
  Ausente numa casca anterior: o miolo confere antes de usar, e segue sem ele.

> **Ainda não providos:** `navigation` (navegação da casca) e `i18n` estiveram previstos neste
> contrato, mas **não existem** na implementação. Um miolo não deve contar com eles. Quando forem
> implementados, entram aqui **e** na casca na mesma mudança.

## Modelo de capacidade

Derivado de **(papéis do token na organização do tenant) × (modelo do tenant)**. Diz **o que pode**;
não diz **como se vê** (isso é o miolo).

```jsonc
{
  "org": "...", "project": "...", "tenantId": "...",
  "aggregates": [
    {
      "aggregate": "...", "boundedContext": "...",
      "commands": [
        {
          "name": "...",
          "alias": "...",
          "roles": ["..."],
          "fromState": ["..."],
          "endState": "...",
          "attributes": [ { "name": "...", "type": "...", "role": "input|status|serverStamp|lookup|computed" } ],
          "valueObjects": [ { "name": "...", "cardinality": "single|multiple",
                              "fields": [ { "name": "...", "type": "...", "role": "..." } ], "scalar": false } ]
        }
      ],
      "events": { "<evento>": { "alias": "..." } }
    }
  ]
}
```

- **`commands`** — **apenas** os autorizados ao papel do usuário (interseção já aplicada pelo BFF).
- **`alias`** e **`events`** — o nome de exibição que o modelo declara para comando e evento; a tela
  mostra o alias e, sem ele, o nome. Os dois são **ausentes** quando o modelo não declara, e `events`
  traz só os eventos que declaram. O comando continua sendo enviado pelo `name`. Detalhe, inclusive como
  casar o evento do histórico: [bff — Nome de exibição](../bff/README.md#nome-de-exibicao).
- **`fromState`** — de quais estados o comando aparece; a casca/miolo habilita a transição só quando o
  agregado está num estado válido.
- **`attributes[].role`** — classificação que guia a apresentação:
  - `input` — campo editável comum.
  - `status` — campo de transição/concorrência (não é entrada do usuário).
  - `serverStamp` — carimbo do servidor: **oculto no formulário e retirado do comando no envio**.
    ⚠️ **É o atributo que o agregado declara como `whenAttribute` de um evento — nunca "Timestamp com
    nome terminado em `em`".** Atributo de data que o negócio calcula, ou que o usuário informa, é
    `input`: viaja normalmente no comando. Só o carimbo declarado fica de fora.
  - `lookup` — referência a outro agregado (widget de seleção).
  - `computed` — o **processor de regra de negócio preenche**: **oculto no formulário**, e o BFF o envia
    com um marcador que o processor sobrescreve. Nasce de declaração no modelo, nunca de nome. Vale
    também para `valueObjects[].fields` — e um value object em que todo campo é `computed` não se
    desenha. Declaração, marcadores e a regra do processor:
    [bff — Atributo calculado pelo br](../bff/README.md#atributo-calculado-pelo-br--computed).
- **`valueObjects`** — os value objects que o comando aceita, **ausente** quando não há nenhum. É a
  segunda metade do que o comando aceita: sem ela, uma tela montada só de `attributes` não oferece o
  campo, e um comando com value object obrigatório fica **impossível de disparar**.
  - **`cardinality` decide a tela**: `single` → um **grupo** de campos; `multiple` → uma **lista**, com
    "adicionar" e "remover". É por isso que ela viaja, em vez dos campos achatados: achatar perderia
    exatamente essa distinção.
  - **`fields` vazio** = o modelo declarou o value object como atributo tipado direto. Não há campos a
    oferecer, e quem preenche precisa conhecer o modelo.
  - **A forma do valor no envio é uma só**, independente de como o modelo declara: `single` é
    **objeto**, `multiple` é **array de objetos**. Escalar é recusado com `400` pela plataforma.

  > **Entrou em 2026-09-13.** Antes o value object viajava no **envio** do comando e não aparecia na
  > capacidade — quem montava tela a partir dela concluía que o campo não existia, sem aviso. Corrigido
  > nos três lados do contrato ao mesmo tempo, como esta seção exige.

## Regras

- **Modelo = domínio (autoritativo)**: existência de comando, papéis, tipos, transições — o miolo **não
  sobrescreve**. **Miolo = apresentação**: rótulo, widget, layout, ordem, visibilidade — o modelo **não
  fornece**, o miolo **preenche**. A exceção é o nome de exibição de comando e evento (`alias`), que o
  modelo pode declarar e o miolo deve respeitar.
- **Nada de segredo no miolo**: toda escrita/leitura de domínio passa pelo `api` → **BFF** → plataforma,
  que **revalida** a autorização no servidor.
- **Ciclo de vida**: `mount` ao selecionar o bounded context; `dispose` ao trocar de BC ou encerrar.
