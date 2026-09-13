# bff — guia do serviço (endpoints e contratos)

> O **BFF** (Backend-for-Frontend) é a **borda de consumo** da plataforma: faz o login, guarda o token
> (cookie httpOnly), **deriva a capacidade** do tenant, **resolve o miolo** por tenant e faz **proxy** ao
> gateway injetando as credenciais **no servidor**. Quem consome fala **só** com o BFF — nenhum segredo
> sai para o cliente.
>
> **Não é exclusivo de frontend.** A casca universal é o consumidor mais comum, mas o BFF atende
> igualmente **cliente sem UI** (sistema backend/API-only, app móvel, integração de terceiro): a
> resolução de miolo simplesmente não é usada, e sessão/capacidade/proxy valem igual. Por isso vive aqui
> como **serviço próprio**, e não sob `shell/`.
>
> Pré: [06-autenticacao](../06-autenticacao.md). Para o uso com casca+miolo:
> [shell/README](../shell/README.md), [shell/seguranca](../shell/seguranca.md).

## Como alcançar o BFF

**O BFF não segue a regra de endereçamento dos oito serviços.** Aqueles ficam atrás do API Gateway e
se resolvem por uma base + o prefixo `/v3/<svc>` ([06-autenticacao](../06-autenticacao.md)). O BFF
**não**:

| | oito serviços da plataforma | BFF |
|---|---|---|
| atrás do gateway | sim | **não** |
| prefixo `/v3/<svc>` | sim | **não** — os paths ficam na raiz |
| registro no service discovery | sim | **não** |
| quantos existem | um de cada, para todos | **um por instalação** |

O motivo é o que a introdução diz: o BFF é **camada de aplicação**, não serviço da plataforma. Cada
instalação sobe o seu, com domínio próprio. Procurá-lo no gateway ou na configuração da plataforma
**não encontra nada** — e isso é o esperado, não um defeito.

**Onde ele está, então:** o endereço é **informado por quem publica a instalação**. Não há como
derivá-lo, nem convenção a adivinhar.

| instalação | endereço |
|---|---|
| `yc.app` (casca universal "Stager") | `https://bff.stager.ycodify.com` |

**Como chamar** — os paths deste documento vão **na raiz**, sem prefixo nenhum:

```
POST https://bff.stager.ycodify.com/session/login     ✅
POST https://api.ycodify.com/v3/bff/session/login     ❌ não existe
```

E **toda** chamada precisa mandar o cookie de sessão (`credentials: 'include'` no browser, jar de
cookies fora dele): a sessão vive num cookie httpOnly, não num cabeçalho. Sem isso, tudo depois do
login responde `401`.

> Os endereços dos **serviços da plataforma** que o BFF consome continuam sendo configuração de deploy
> e não constam aqui — ver o operacional em `yc.app/docs`.

## Sessão

| Operação | Método · Path | Corpo / Query | Resposta |
|---|---|---|---|
| Login | `POST /session/login` | `{ username, password }` | identidade + **árvore de tenants** + papéis; **cookie httpOnly** (token). **Sem token no corpo.** |
| Sessão atual | `GET /session/me` | — | mesma projeção segura, ou `401` sem sessão |
| Papel ativo | `PUT /session/active-role` | `{ org, role }` | mesma projeção segura, com `activeRole` · `403` se o papel não for do usuário naquela org |
| Encerrar | `POST /session/logout` | — | limpa o cookie |

`POST /session/login` chama o [auth](../auth/README.md) (`/ua/sign-in`), guarda o token e devolve **só a
projeção segura**:

```jsonc
{
  "identity":   { "username": "...", "name": "...", "email": "...", "status": "...", "exp": 0 },
  "orgs":       [ { "owner": "...", "name": "...", "alias": "...", "status": "..." } ],
  "roles":      [ { "role": "...", "org": "...", "status": "...", "alias": "..." } ],
  "tenants":    [ /* tuplas org:project:boundedContext:dataschema:tenant */ ],
  "tenantTree": [ { "org": "...", "projects": [ { "project": "...", "boundedContexts": [ { "boundedContext": "...", "tenantId": "..." } ] } ] } ],
  "activeRole": null
}
```

**`activeRole`** é o papel que o usuário escolheu **para esta sessão** — serve a quem tem mais de um
papel na org e entra por um portal específico. É **filtro de apresentação, não autorização**: o BFF
aceita apenas um papel que o portador de fato tenha **ativo naquela org**, e comando/consulta seguem
revalidando o conjunto **completo** de papéis a cada chamada. Guardá-lo não restringe o que o usuário
pode fazer.

## Capacidade

| Operação | Método · Path | Resposta |
|---|---|---|
| Capacidade do tenant | `GET /session/capabilities?tenant={tenantId}` | **modelo de capacidade** (ver [contrato-miolo](../shell/contrato-miolo.md#modelo-de-capacidade)) |

O BFF lê o **modelo publicado** do tenant no **serviço de cache** — **não** no forger (o
forger é quem **grava** ali ao publicar o `.model.json`) — e cruza com os **papéis org-scoped** do token
→ devolve **só** os comandos autorizados. Só para tenant que **consta no token**; senão `404`.

**Cada comando descreve o que aceita em duas listas**, e quem monta tela precisa das duas:

| Chave | O que traz |
|---|---|
| `attributes` | os atributos escalares, com `type`, `nullable` e o papel de apresentação (`input`, `status`, `serverStamp`, `lookup`) |
| `valueObjects` | os **value objects** do comando, com `name`, `cardinality` (`single` · `multiple`) e `fields`. **Ausente** quando o comando não declara nenhum |

A `cardinality` é o que decide a tela — `single` desenha **um grupo**, `multiple` desenha **uma lista**
com "adicionar" — e é por isso que ela viaja em vez dos campos achatados. `fields` vazio significa que o
modelo declarou o value object como atributo tipado direto: não há campos a oferecer, e quem preenche
precisa conhecer o modelo.

> **Errata, 2026-09-13.** Até esta data `valueObjects` **não existia na capacidade**, embora o comando
> aceitasse o campo no envio. Quem montava tela a partir da capacidade concluía que o campo não existia,
> sem nenhum aviso — e o **miolo genérico não conseguia disparar comando com value object obrigatório**,
> por montar o formulário só de `attributes`. Era omissão por custo, registrada em comentário no código
> desde setembro: expor exigia mudar o contrato pareado nos três lugares de uma vez. Medido de fora pelo
> `clubflow` (`yc.app/issues/bff.bug.capacidade-nao-expoe-valueobject-do-comando.20260913.md`), que
> comparou o modelo publicado com a capacidade devolvida.

> ⚠️ **A capacidade depende do modelo estar VIVO no cache**, e quando ele some a causa é **remoção,
> nunca expiração.** Sem a chave, o cache responde `204`, a capacidade do bounded context **some** —
> `GET /session/capabilities` deixa de listar os comandos daquele tenant — e isso acontece **mesmo com o
> sistema provisionado e o dataschema `RUNNING`**. Não é falha do BFF: é o modelo que saiu do cache.
> Vale para **qualquer** consumidor — casca ou cliente sem UI.
>
> **O modelo publicado NÃO tem prazo de validade.** O forger grava a chave **sem expiração**, e isso é
> deliberado: a constante que carrega o valor existe *"para que nenhuma configuração externa possa
> reintroduzir TTL no modelo publicado"* — não é propriedade de deploy, é constante de código. *(medido
> pelo `composer` em `forger@421a8b8`, `ModelCacheService` — a classe que grava a chave do write model,
> que é a que o BFF lê; a do read model, `EntitiesModelCacheService`, faz igual. Confirmado pelo dono em
> 2026-09-10. O mesmo consta em
> [persistence-q — pré-requisitos do chamador](../persistence-q/README.md): a entrada "não expira
> sozinha".)*
>
> **São DUAS chaves no cache, e a capacidade depende de uma só.** O BFF lê
> `ENGINE:persistence:cqrs:SETUP-TO:<tenantId>:wm` — o **write model**, gravado ao publicar o
> `.model.json`. A outra, `ENGINE:persistence:SETUP-TO:<tenantId>`, guarda a **spec de entidades** (read
> model) e é a que o **motor** usa. Confundi-las manda o diagnóstico para o lado errado, porque **o que
> apaga uma não apaga a outra**.
>
> **O que apaga a chave das capabilities é o `DELETE` explícito do modelo** — `ModelController
> .deleteModel`, que chama `modelCacheService.delete(tenantId)`. Só isso. Fora esse caminho nada a
> remove, e republicar o `.model.json` (`POST forger .../tenant/<id>/model`) a sobrescreve — o mesmo
> `update` cobre o caso de ela estar ausente. Então, se a capacidade sumiu, as hipóteses são **duas**:
> o modelo foi apagado, ou nunca foi publicado para aquele tenant.
>
> **O bracket do dataschema NÃO derruba a capacidade** — ele age sobre a outra chave. A transição
> `RUNNING → MODELING` remove a spec de entidades e com isso bloqueia o **motor**, que passa a responder
> `510: Modelo do tenant não encontrado na cache. Republique o modelo.`; o caminho de volta,
> `MODELING → RUNNING`, a grava de novo. É salvaguarda deliberada. Sintoma diferente, chave diferente:
> `GET /session/capabilities` continua listando os comandos enquanto o comando falha no motor.
>
> **A pergunta certa, portanto, não é "há quanto tempo não se republica?"** — é "este modelo foi
> apagado, ou nunca foi publicado?". Republicar resolve em qualquer hipótese, e é por isso que o
> diagnóstico errado sobrevivia: a remediação funciona mesmo quando a explicação está trocada.
> Diagnóstico que culpa o tempo faz **esperar**; diagnóstico que culpa a remoção faz **investigar** — e
> investigar a chave errada custa quase o mesmo que esperar.
>
> > **Errata, 2026-09-10 — e a parte instrutiva não é o erro, é como ele nasceu.** Até hoje o parágrafo
> > afirmava que o modelo publicado tinha TTL e que o prazo era configuração de deploy do forger.
> > **As duas coisas eram VERDADE quando foram escritas:** até `forger@d9e26f0` (2026-08-31) o
> > `ModelCacheService` declarava `@Value("${app.model.cache.expires:86400}")` — 24 horas por padrão, e
> > o prazo era mesmo config de deploy. Naquele commit o forger passou a gravar sem prazo e a
> > propriedade foi removida; **esta fatia não foi atualizada junto, e apodreceu por catorze dias.**
> >
> > O risco que isto expõe é estrutural, e não se resolve conferindo melhor: **doc de comportamento
> > alheio envelhece quando o dono do comportamento muda sem avisar quem documentou.** Quem escreve
> > sobre serviço de outro não tem como saber que precisa reconferir — foi o `composer` quem mediu a
> > história do arquivo e trouxe a data. *(medição do `composer`; apontada em
> > `yc.app/issues/docs.bug.bff-afirma-que-o-modelo-publicado-tem-ttl.20260910.md`)*
> >
> > **Segunda rodada, no mesmo dia.** A primeira correção trocou o TTL pelo bracket do dataschema — e
> > errou de chave, mandando investigar o `MODELING` de um modelo cuja remoção não passa por ali. O
> > `composer` mediu as duas chaves e desfez a confusão: o `ModelCacheService` grava o write model (o
> > que o BFF lê) e o `EntitiesModelCacheService` grava o read model (o que o bracket remove).
> >
> > **A história inteira deste parágrafo, que é o que vale guardar:** uma afirmação **correta** que
> > apodreceu quando o comportamento mudou sem aviso; uma correção que acertou o "não tem TTL" e errou
> > de chave; e só então a certa. Nenhuma das três falhava na prática, porque a remediação — republicar
> > — funciona em todas as hipóteses. **Parágrafo cuja receita sempre dá certo não avisa quando a
> > explicação está errada**, e este já demonstrou isso três vezes.

## Miolo

| Operação | Método · Path | Resposta |
|---|---|---|
| Manifesto do miolo | `GET /tenant/{tenantId}/miolo-manifest` | `{ "tenantId": "...", "manifestUrl": "..." }` |

Resolve `tenant → URL` do miolo (ver [injecao](../shell/injecao.md)). Só para tenant do usuário.

> **Única operação exclusiva de quem usa a casca.** Consumidor **sem UI** (app móvel, integração,
> serviço) **não** chama este endpoint — não há miolo a injetar. Sessão, capacidade e proxy valem
> integralmente para ele.

## Proxy de domínio

O BFF expõe escrita e leitura de domínio ao consumidor (no caso da casca, via `api` do hostContext),
fazendo **proxy** ao [persistence-crs](../persistence-crs/README.md) e ao
[persistence-q](../persistence-q/README.md) — **injetando `Authorization` + `X-Tenant-Id` no servidor**.
O consumidor **não** compõe esses cabeçalhos nem conhece o token.

| Operação | Método · Path | Corpo | Vai para |
|---|---|---|---|
| Executar comando | `POST /session/command` | `{ tenantId, aggregate, command, data, id?, status? }` | persistence-crs (escrita) |
| Consultar projeção | `POST /session/query` | `{ tenantId, aggregate, predicates?, paging?, sorting? }` | persistence-q (leitura) |
| Estado atual do agregado | `POST /session/aggregate` | `{ tenantId, aggregate, id }` | persistence-crs (leitura) |
| Histórico do agregado | `POST /session/history` | `{ tenantId, aggregate, id }` | persistence-crs (`/history`) |

- `aggregate` é `"<boundedContext>.<tipo>"`; `id` é o **UUID do agregado** (`aggregateid` da projeção),
  nunca a PK `Long` da linha — ver [antipadrões](../05-antipatterns.md).
- O BFF **re-deriva a capacidade** antes de encaminhar um comando e recusa (`403`) o que não estiver
  autorizado ao papel; o persistence-crs revalida de novo. Ver [seguranca](../shell/seguranca.md).
  **Isso vale para o comando. Leitura é outra história — ver abaixo.**
- **Value object viaja no comando.** `data` carrega, no **mesmo nível** dos atributos escalares, cada
  `valueObject` declarado pelo comando no modelo (ver
  [model-format](../persistence-crs/spec/model-format.md#datavalueobject)): `single` → **um objeto**;
  `multiple` → **um array de objetos**. **Vale para as duas formas de declaração** (grupo de campos e
  atributo tipado direto): a forma do **valor** é a mesma nas duas, e **escalar nunca** — nem solto,
  nem dentro de array. Forma incompatível é recusada com `400` que **diz a posição** do item errado,
  **nunca omitida em silêncio** (um item fora da forma reprova o comando inteiro). O BFF não coage os
  campos internos: repassa o valor como recebeu, aninhamento incluso.
- `/session/aggregate` existe porque a **projeção é assíncrona**: para carregar o estado autoritativo de
  um agregado (ex.: preencher um form de transição) não se deve ler o read model.

### Quem autoriza a leitura

Escrita e leitura **não** são governadas do mesmo jeito, e a diferença é deliberada:

| Rota | Quem decide | Como |
|---|---|---|
| `POST /session/command` | BFF **e** persistence-crs | papéis do token × `command.roles` do modelo; recusa `403` |
| `POST /session/query` | **persistence-q**, sozinho | `_conf.accessControl.read` da entity |
| `POST /session/aggregate` · `/session/history` | BFF | mesma derivação de capacidade do comando |
| **quais COLUNAS**, nas três de leitura | **BFF** | `readProjection` do agregado — ver [abaixo](#limitar-os-campos-que-um-papel-lê--readprojection) |

`POST /session/query` **não aplica portão de papel**. O BFF confere que o `tenantId` pertence ao
portador e encaminha; quem autoriza é o persistence-q, pela política de leitura declarada na entity.
A razão é de domínio: **ler não é executar**. Amarrar leitura aos papéis dos comandos tornava
impossível um papel **somente-leitor** — para ver um catálogo, alguém teria de ganhar um comando que
não deveria ter, falsificando o modelo e afrouxando a escrita.

> ⚠️ **O default é restritivo.** Entity **sem** `_conf.accessControl.read` declarado, ou com a lista
> vazia, **não é legível** — o persistence-q recusa. Autorizar leitura é ato explícito: declarar o
> papel na política da entity, o que vale de imediato, sem republicar modelo nem reiniciar serviço.

`/session/aggregate` e `/session/history` **mantêm** o portão de capacidade porque batem no
persistence-crs, que valida **tenant**, não leitura por papel — sem o portão, estado e histórico de
qualquer agregado ficariam ao alcance de qualquer portador de sessão daquele tenant.

### Limitar os campos que um papel lê — `readProjection`

O `accessControl` da plataforma recorta **linha**: `read` libera a entity inteira e `scope` limita às
linhas de que o papel é titular. **Nada limita coluna.** Para um papel que deve ver parte de uma
ficha — a recepção que precisa de nome e telefone e nunca de CPF — sobravam dois extremos: ler tudo
ou não ler nada.

O BFF preenche essa lacuna com `readProjection`, **declarativo e por papel**, válido para qualquer
entity.

**Onde se declara:** no `.model.json` do tenant, **ao lado dos comandos do agregado**.

```jsonc
"pessoal.aluno": {
  "command": { … },
  "readProjection": { "RECEPCIONISTA": ["nome", "email", "telefone", "genero"] }
}
```

**A declaração é EXAUSTIVA por agregado.** Lista-se todo papel que pode ler, e `"*"` é como se
declara "este papel vê a ficha inteira":

```jsonc
"readProjection": {
  "RECEPCIONISTA": ["nome", "email", "telefone", "genero"],
  "ADMINISTRADOR": "*"
}
```

| Situação | Lê |
|---|---|
| o agregado **não** declara `readProjection` | a linha inteira |
| declara, e **nenhum** papel do usuário está na lista | a linha inteira |
| declara, e **algum** papel do usuário está na lista | **só** a união das colunas dos papéis declarados |
| papel listado com `"*"` | a linha inteira |

⚠️ **Papel do usuário que não está na declaração é IGNORADO — ele não alarga o recorte.** Se um papel
deve ver a ficha inteira, **declare-o com `"*"`**; não basta omiti-lo.

> **Errata, 2026-09-13, e ela vale como aviso de modelagem.** Até esta data a regra era a do `scope`
> de linha — *"vários papéis, um deles fora → lê tudo"*. Aquilo é correto quando o outro papel **de
> fato lê** a entity, e o BFF **não tem como saber isso**: quem concede leitura é o
> `accessControl.read` do `_conf`, que ele não enxerga. Medido com conta real: um usuário
> `[VISITANTE, RECEPCIONISTA]` recebia a ficha inteira **com CPF**, porque `VISITANTE` não estava na
> declaração — **embora `VISITANTE` não tivesse leitura nenhuma naquela entity**. Como quase todo
> usuário acumula papéis, o recorte quase nunca disparava e o controle era praticamente inerte.
>
> A regra nova **falha fechando**: esquecer o `"*"` de um papel faz ele perder colunas, o que aparece
> no primeiro uso — em vez de vazar dado pessoal em silêncio.

**Três colunas nunca se recortam:** `id`, `aggregateid` e `status`. Sem elas a tela não seleciona
registro nem sabe que transições cabem, e cortá-las não protegeria dado pessoal nenhum — são
identificador e estado.

**Vale nas três rotas de leitura**, não só na consulta: `POST /session/query`,
`POST /session/aggregate` e `POST /session/history`. É o mesmo dado por três portas — no histórico o
recorte cai sobre o `eventData` do evento, e os metadados (quem, quando, qual comando) ficam.

> **Nas duas últimas, o recorte é a SEGUNDA barreira, não a primeira.** `aggregate` e `history`
> mantêm o **portão de capacidade**: papel sem comando no agregado recebe `403` e não chega ao
> recorte. Então, para um papel estreito — que tipicamente não dispara comando ali —, quem fecha a
> porta é o portão, e o recorte só entra em cena quando o papel **tem** comando no agregado **e** está
> declarado. Medido com conta real em 2026-09-13: o `RECEPCIONISTA` recebe `403` nas duas, e por isso
> o recorte **não foi exercitado** por esse caminho. Não confunda porta fechada com campo recortado —
> a primeira é o portão fazendo o trabalho, e ela some assim que o papel ganhar um comando.

> ⚠️ **A plataforma não valida a declaração, e o BFF valida.** O forger aceita com `201` papel
> inexistente, coluna com nome errado e lista vazia: ele confere `aggregate`, `org/project/tenant` e
> os placeholders, e carrega o resto sem olhar. Então **declaração inválida faz a leitura ser recusada
> com `500`**, nomeando o que está errado — não recortada pela metade. Entregar tela que parece
> funcionar sobre uma regra de acesso que não se sustenta é pior que recusar.
>
> **Nunca prefixe a chave com `_`.** A publicação do `.model.json` remove chaves `_`-prefixadas em
> qualquer profundidade, **em silêncio** — a declaração sumiria sem aviso nenhum.

> ### ⚠️ Esta chave viaja por uma propriedade MEDIDA, não por contrato
>
> O `.model.json` é gravado no cache **como recebido**, menos as `_`-prefixadas — é isso que faz uma
> chave desconhecida como `readProjection` chegar ao BFF. Mas **ninguém no forger prometeu preservá-la,
> e nenhum teste a protege**: é comportamento observado, não garantia. Se a publicação de modelo um dia
> ganhar validação de esquema, ou um gerador de vocabulário fechado como o que já existe do lado da
> entity, a declaração **desaparece em silêncio** e o recorte deixa de valer sem nada acusar.
>
> Dito pelo `composer`, dono do forger, em 2026-09-13, corrigindo uma formulação anterior desta doc que
> tratava a sobrevivência como se fosse promessa. **Quem for mexer naquele caminho precisa saber que
> ele carrega política de acesso a dado pessoal.**

> ### 🔴 O que `readProjection` NÃO faz
>
> **Só alcança quem passa pelo BFF.** O motor não conhece esta declaração: ela é metadado que só o
> BFF lê. Quem chamar o `persistence-q` **direto**, com o próprio token, recebe a linha inteira — e
> isso não é hipótese, é prática medida em 2026-09-08 e reconfirmada em 09-09 num sistema que usa
> esta plataforma.
>
> Em uma frase: **`readProjection` tira o campo da tela, não do banco.** Enquanto o recorte de coluna
> não existir no `persistence-q`, quem precisa da garantia no dado tem de negar a leitura da entity
> inteira no `accessControl.read`, ou separar o dado sensível em outra entity.

### `predicates` é a forma do persistence-q — não há dialeto do BFF

`predicates` é o **objeto de predicados** do
[persistence-q](../persistence-q/endpoints/consulta.md), repassado como está. O BFF só acrescenta o
**rótulo** (o nome da entity, derivado de `aggregate`) e os controles; ele **não traduz** nada.

```json
{
  "tenantId": "…", "aggregate": "pessoal.associado",
  "predicates": { "status": { "neq": "encerrado" } },
  "paging":  { "_maxRegisters": 50, "_firstRegister": 0 },
  "sorting": { "_orderBy": "criadaem", "_order": "DESC" }
}
```

- Cada predicado é `{ "<atributo>": "<valor>" }` (igualdade) ou `{ "<atributo>": { "<op>": "<valor>" } }`,
  com `<op>` em `eq/neq/gt/gte/lt/lte/like/ilike/in` — **minúsculas**. O vocabulário de atributos é o do
  modelo provisionado; nome desconhecido é rejeitado pelo persistence-q (`510`).
- ⚠️ **`predicates` é objeto. Array é recusado com `400`** e mensagem dizendo a forma esperada. Uma
  lista de descritores (`[{attribute, operator, value}]`) **não** é aceita: espalhá-la geraria a chave
  `"0"`, que o persistence-q leria como atributo do modelo — rejeição indecifrável no melhor caso,
  filtro silenciosamente diferente do pedido no pior.
- `paging`/`sorting` são conveniências do BFF, que os coloca no **nível raiz** do critério (irmãos do
  rótulo) e indexa `sorting` por posição, como exige
  [query-controls](../persistence-q/query-controls.md). O consumidor envia a forma plana acima.
- ⚠️ **Sem `paging`, a lista vem cortada e o consumidor não fica sabendo.** O read model aplica um
  **limite padrão** (500 registros no modo array) — a resposta parece completa. Quem lista **manda
  `paging` sempre**. E **meio `paging` é pior que nenhum**: incompleto, a consulta roda **sem limite**
  e devolve a tabela inteira do tenant, então o BFF recusa a forma incompleta com `400` antes de ir à
  rede. O mesmo vale para `sorting` sem `_orderBy` ou com `_order` fora de `ASC`/`DESC`.
- **A resposta diz quando cortou.** Havendo mais registros além do teto, o sucesso traz
  `truncated: true` e `maxRegisters` ao lado de `records`. **Ausência de `truncated` não é garantia de
  lista inteira** enquanto a marca não estiver no ar em toda a plataforma — só a presença é
  informação. Para continuar, repita a consulta avançando `_firstRegister` em `_maxRegisters`.
- **O rótulo é a chave que NÃO começa com `_`.** Quem procura o resultado pela "primeira chave do
  item" da resposta do persistence-q passa a tropeçar em `_truncated` e a ler um booleano como lista.
- **Erro do upstream chega inteiro.** Em `/session/query`, `/session/aggregate` e `/session/history`, um
  não-2xx do persistence-q/-crs é repassado com **o status e o corpo de erro reais**. O consumidor
  **nunca** recebe status de erro com corpo em forma de sucesso — `{ records: [] }`, `{ state }` e
  `{ events }` só existem no caminho de sucesso, e `204` (vazio) continua virando `200` com lista vazia.

## Autocadastro (`/ua/*`)

Fluxo público de **auto-registro de conta externa**, proxy dos endpoints `/open/ua/*` e `/ua/open/*` do
[orgid](../orgid/README.md). Sem `Authorization` — o BFF injeta `X-Forger-Credential` no servidor.
Contrato de origem: [orgid/publico](../orgid/endpoints/publico.md) e [orgid/ua-papel](../orgid/endpoints/ua-papel.md).

| Operação | Método · Path | Corpo / Query | Resposta |
|---|---|---|---|
| Papéis oferecíveis | `GET /ua/roles?owner={org}` | — | `{ roles: [{ name, label }] }` — só `ispublic=true` |
| Conta já existe? | `GET /ua/exists?username={u}` | — | `{ exists: true\|false }` |
| Registrar | `POST /ua/register` | `{ account: {username,password,email,name?}, role: {name,owner} }` | `200` criado · `403` papel fora do cardápio · `409` papel inexistente |
| Pedir hash (e-mail) | `GET /ua/hash?action={R\|PR}&username={u}` | — | repassa o orgid |
| Ativar / recuperar | `PUT /ua/activate` | `{ username, action: R\|PR, hash, password? }` | repassa o orgid |

> ⚠️ **Duas travas ficam no BFF, porque o orgid não as faz.**
> 1. **O papel é validado contra o cardápio público** (`GET /ua/roles`, que lista só `ispublic=true`).
>    O orgid **ignora** `role.ispublic` no corpo e associa **qualquer papel existente** — sem esta
>    checagem, um registro público pediria um papel privilegiado e o receberia. Fora do cardápio → `403`.
> 2. **A conta nasce sempre `PENDING`.** O contrato do orgid aceita `account.status: ACTIVE` (e a chave
>    `from` na raiz) para pular a ativação; o BFF força `PENDING` e não repassa `from`, de modo que só o
>    fluxo de hash por e-mail ativa a conta.
>
> O `204` do orgid **não é sucesso** (significa papel inexistente e nada criado, transação revertida) —
> o BFF o converte em `409` para não ser lido como ok.

Monte a tela de cadastro a partir de `GET /ua/roles`: são exatamente os papéis que o servidor aceita.

## Preferências da organização (org-scoped)

Preferências de **apresentação por organização** — valem para **todos** os usuários da org, persistidas no
BFF. Leitura por qualquer membro **ativo**; escrita **só por MASTER-na-org** (revalidada no servidor).

| Operação | Método · Path | Corpo / Query | Resposta |
|---|---|---|---|
| Ler prefs | `GET /session/org-preferences?org={org}` | — | `{ org, prefs, canConfigure }` |
| Alterar prefs | `PUT /session/org-preferences` | `{ org, formMode?, colorScheme? }` (**patch parcial**) | `{ org, prefs, canConfigure: true }` |

```jsonc
// prefs
{ "formMode": "inline" | "modal",           // layout do form de comando no miolo
  "colorScheme": "verde" | "azul" | "ambar" | "roxo" | "cinza" }  // esquema de cor da casca (ver ../shell/estilo.md)
```

- **Patch parcial**: o `PUT` aceita `formMode` e/ou `colorScheme`; campos ausentes mantêm o valor atual.
  Valores fora do catálogo → `400`. `canConfigure` = o portador é MASTER **naquela** org.
- O catálogo de `colorScheme` é **contrato** — espelhado no shell (catálogo/UI) e no BFF (validação).
  Ver [estilo](../shell/estilo.md#esquemas-de-cor-nomeados-paleta-completa-por-organização).

## Cabeçalhos que o BFF injeta (contrato de saída)

Tudo composto **no servidor**. Nenhum deles é montado — nem visto — pelo browser ou pelo miolo.

| Cabeçalho | Em quê | De onde vem |
|---|---|---|
| `Authorization: Bearer …` | comando, consulta, agregado, histórico, capacidade | token da sessão (no store server-side) |
| `X-Tenant-Id` | comando, consulta, agregado, histórico | tenant da requisição, conferido contra os tenants do token |
| `X-Forger-Credential` | **login** (`/ua/sign-in`) e **autocadastro** (`/ua/*`) | credencial de gateway, de configuração de deploy |

> **`X-Forger-Credential` é pré-autenticação.** O gateway a exige **antes** de existir token — o próprio
> `sign-in` passa por ele (ver [06-autenticacao](../06-autenticacao.md)). Por isso o BFF precisa dela em
> configuração: **sem ela, o login falha com `401` no gateway**, e não por credencial de usuário inválida.
> É segredo: nunca vai ao browser, nunca em código, nunca em doc.

## Operação

| Operação | Método · Path | Resposta |
|---|---|---|
| Saúde do serviço | `GET /health` | `{ ok: true, service: "yc-app-bff" }` |

**Aberta, sem cookie** — é a única rota fora de `/ua/*` que não exige sessão. Serve ao healthcheck do
container e não diz nada sobre a plataforma atrás: `200` aqui significa que **o BFF** está de pé, não
que o gateway, o cache ou o persistence respondem.

## Erros

| HTTP | Quando |
|---|---|
| `400` | falta parâmetro (ex.: `username`/`password`, `tenant`), ou campo obrigatório do comando ausente |
| `401` | sem sessão / sessão inválida / credencial inválida — ver `reason` abaixo |
| `403` | comando não autorizado ao papel · papel fora do cardápio público no autocadastro · papel que não é do usuário na org (`active-role`) |
| `404` | tenant não pertence ao usuário / miolo não registrado / **modelo do tenant ausente do cache** (removido ou nunca publicado — ele **não expira**) |
| `409` | autocadastro: papel inexistente — nada foi criado (o `204` do orgid, traduzido) |
| `500` | configuração ausente no servidor (ex.: o path do endpoint de cache não configurado) · **`readProjection` inválido no modelo do tenant** — a leitura é recusada em vez de recortada pela metade |
| `502` | falha ao falar com um serviço da plataforma |

### `401` diz **por que** não há sessão

O corpo do `401` de sessão traz `reason`, para que o consumidor distinga o que antes era indistinguível:

| `reason` | Significado | O que o consumidor deve fazer |
|---|---|---|
| `no-session` | não há sessão — ninguém entrou ainda | pedir login, normalmente |
| `expired` | o prazo da sessão acabou | pedir login de novo |
| `evicted` | a sessão **desapareceu do servidor antes de expirar** | pedir login **e preservar o destino**: o portador não fez nada errado |
| `unknown` | não foi possível determinar | tratar como `expired` |

`evicted` é o caso que não pode ficar mudo: o portador estava trabalhando e perdeu a sessão sem causa
atribuível a ele. Uma interface que trate os quatro como o mesmo `401` seco devolve o usuário ao início
sem explicação — e ninguém consegue medir a frequência do problema.

## Checklist do agente

- [ ] O browser fala **só** com o BFF — nunca com a plataforma direto.
- [ ] O token **nunca** vai no corpo — só em **cookie httpOnly**.
- [ ] `Authorization`, `X-Tenant-Id` **e `X-Forger-Credential`** são compostos **no BFF**
      (ver [seguranca](../shell/seguranca.md)).
- [ ] No autocadastro, ofereça só o que `GET /ua/roles` devolve — o servidor recusa o resto.
- [ ] Endereços de serviço = **config de deploy**, nunca hardcode nem em doc pública.
