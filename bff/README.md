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

## O que está aqui

- [Como alcançar o BFF](#como-alcançar-o-bff)
- [Sessão](#sessão)
- [Capacidade](#capacidade)
- [Miolo](#miolo)
- [Proxy de domínio](#proxy-de-domínio)
- [Arquivos anexos de agregado (`/session/files/*`)](#arquivos-anexos-de-agregado-sessionfiles)
- [Quem autoriza a leitura](#quem-autoriza-a-leitura)
- [Limitar os campos que um papel lê — `readProjection`](#limitar-os-campos-que-um-papel-lê-readprojection)
- [Atributo calculado pelo br — `computed`](#atributo-calculado-pelo-br--computed)
- [`predicates` é a forma do persistence-q — não há dialeto do BFF](#predicates-é-a-forma-do-persistence-q-não-há-dialeto-do-bff)
- [Autocadastro (`/ua/*`)](#autocadastro-ua)
- [Preferências da organização (org-scoped)](#preferências-da-organização-org-scoped)
- [Cabeçalhos que o BFF injeta (contrato de saída)](#cabeçalhos-que-o-bff-injeta-contrato-de-saída)
- [Operação](#operação)
- [Erros](#erros)
- [Checklist do agente](#checklist-do-agente)

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
| `attributes` | os atributos escalares, com `type`, `nullable` e o papel de apresentação (`input`, `status`, `serverStamp`, `lookup`, `computed`) |
| `valueObjects` | os **value objects** do comando, com `name`, `cardinality` (`single` · `multiple`) e `fields`. **Ausente** quando o comando não declara nenhum |

**`serverStamp` é o `whenAttribute` do evento, e só ele.** O BFF **não envia** esse atributo no comando,
porque a plataforma o valoriza sozinha (canon
[model-format](../persistence-crs/spec/model-format.md#eventos-e-domainbus)). Qualquer outro `Timestamp`
declarado em `data.attribute` é `input` e **viaja normalmente** — inclusive o que tem nome terminado em
`em`. ⚠️ **O sufixo do nome não classifica nada**: instante calculado pela regra de negócio é dado do
comando e chega ao motor como qualquer outro.

A `cardinality` é o que decide a tela — `single` desenha **um grupo**, `multiple` desenha **uma lista**
com "adicionar" — e é por isso que ela viaja em vez dos campos achatados. `fields` vazio significa que o
modelo declarou o value object como atributo tipado direto: não há campos a oferecer, e quem preenche
precisa conhecer o modelo.

> ⚠️ **A capacidade depende do modelo estar VIVO no cache**, e quando ele some a causa é **remoção,
> nunca expiração.** Sem a chave, o cache responde `204`, a capacidade do bounded context **some** —
> `GET /session/capabilities` deixa de listar os comandos daquele tenant — e isso acontece **mesmo com o
> sistema provisionado e o dataschema `RUNNING`**. Não é falha do BFF: é o modelo que saiu do cache.
> Vale para **qualquer** consumidor — casca ou cliente sem UI.
>
> **O modelo publicado NÃO tem prazo de validade.** O forger grava a chave **sem expiração**, e isso é
> deliberado: a constante que carrega o valor existe *"para que nenhuma configuração externa possa
> reintroduzir TTL no modelo publicado"* — não é propriedade de deploy, é constante de código. *(medido
> em `forger@421a8b8`, no serviço que grava a chave do **write model** — a que o BFF lê; o serviço do
> **read model** faz igual. Confirmado pelo dono em
> 2026-09-10. O mesmo consta em
> [persistence-q — pré-requisitos do chamador](../persistence-q/README.md): a entrada "não expira
> sozinha".)*
>
> **São DUAS chaves no cache, e a capacidade depende de uma só.** ⚠️ **As duas aparecem MASCARADAS
> abaixo:** o prefixo interno da plataforma está substituído por `<prefixo-interno>`, e o formato literal
> **não é este** — o que a página afirma é a **distinção** entre elas, não o valor. O BFF lê
> `<prefixo-interno>:cqrs:<tenantId>:wm` — o **write model**, gravado ao publicar o `.model.json`. A
> outra, `<prefixo-interno>:<tenantId>`, guarda a **spec de entidades** (read model) e é a que o **motor**
> usa. Confundi-las manda o diagnóstico para o lado errado, porque **o que apaga uma não apaga a outra**.
>
> **O que apaga a chave das capabilities é o `DELETE` explícito do modelo** — o controlador de modelo do
> forger manda o serviço de cache remover a entrada daquele tenant (nomes de classe e método **omitidos**:
> são implementação, não contrato). Só isso. Fora esse caminho nada a
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
> > *Este parágrafo foi corrigido três vezes em 2026-09-10 — ver [histórico de correções](#histórico-de-correções).*

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
- **Valor fora do tipo declarado é recusado com `400`, nunca trocado.** `Boolean` aceita `true`/`false`
  (ou os textos `"true"`/`"false"`); `Integer`/`Long` aceitam número inteiro (ou texto só de dígitos).
  `"sim"`, `"1"`, `"abc"` e `"1,5"` são recusados, com a mensagem nomeando o campo. `""` e `null` seguem
  contando como ausentes. As datas vão como chegam, e o persistence-crs as normaliza ou recusa.
- **Atributo `computed` não é entrada de ninguém.** O BFF o envia sempre com um marcador explícito e
  não nulo, e o processor br o sobrescreve; o consumidor não precisa mandá-lo, e o que mandar é ignorado — ver
  [Atributo calculado pelo br](#atributo-calculado-pelo-br--computed).
- `/session/aggregate` existe porque a **projeção é assíncrona**: para carregar o estado autoritativo de
  um agregado (ex.: preencher um form de transição) não se deve ler o read model.

## Arquivos anexos de agregado (`/session/files/*`)

O BFF faz **proxy do [filer](../filer/README.md)**, o serviço de arquivos da plataforma, pelas mesmas
regras das outras rotas de sessão: o consumidor manda o **cookie**, e o BFF injeta `Authorization` e
`X-Tenant-Id`. O arquivo é **anexo de um agregado** e se identifica só pela chave
`{org}-{project}-{entity}-{entityId}-{attribute}.{ext}`, em que **`entityId` é o `aggregateid`** — o
mesmo UUID das rotas de domínio, nunca a PK da projeção.

| Operação | Método · Path | Parâmetros | Corpo |
|---|---|---|---|
| Enviar | `POST /session/files/upload` | `tenantId`, `filename` (query) | binário, `Content-Type: application/octet-stream` |
| Baixar | `GET /session/files/download` | `tenantId`, `filename` (query) | — |
| Listar | `GET /session/files/list` | `tenantId`, `filename` = **prefixo** (query) | — |
| Remover | `DELETE /session/files/delete` | `tenantId`, `filename` (query) | — |

- **Upload e download exigem a chave completa**, com extensão. **Listar e remover aceitam prefixo
  parcial, e prefixo significa LOTE** — remover por `…-{entity}-{entityId}-` apaga **todos** os anexos
  daquele agregado, e por `…-{entity}-` todos os do tipo.
- Respostas: `list` devolve `{ files: [...] }` (embrulhado, para poder crescer sem quebrar quem
  consome); `delete` devolve o relatório do filer, `{ deleted, prefix, files }` — **prefixo sem
  correspondência é `deleted: 0` com `200`**, não erro; `download` devolve o binário com o
  `Content-Disposition` que o filer escolheu.
- **Erro do filer passa inteiro** — status e corpo reais, como nas rotas de domínio.

> ⚠️ **É proxy PURO: quem autoriza é o filer, e o que ele confere é o papel na entity INTEIRA.**
> O BFF checa a sessão e que o `tenantId` pertence ao portador — **não** checa se o anexo é *daquela
> pessoa*. O filer aplica `accessControl.read` (baixar/listar) e `write` (enviar/remover) do agregado;
> ele **não** aplica o [`accessControl.scope`](../persistence-q/README.md#recorte-de-leitura-por-titular),
> que é o que recorta linha por titular. **Consequência prática: um papel com leitura na entity baixa o
> anexo de qualquer titular, bastando saber o `aggregateid`** — e `list` por prefixo os enumera.
> Anexo que for dado sensível **não** se protege pelo papel: ou o modelo põe o arquivo num agregado de
> acesso mais estreito, ou o titular precisa ser parte da chave de autorização, e isso é decisão de
> quem modela o tenant — não do BFF.

- **Teto por arquivo: 3 MiB** (`3145728` bytes) — acima disso a recusa vem antes de o conteúdo subir.
  É o `max-file-size` do próprio serviço de arquivos, lido em unidade binária.
- **Extensões aceitas:** a whitelist do [filer](../filer/README.md#limites-e-tipos). Fora da lista,
  `400`; acima do teto, `413`.
- **O endereço do filer é configuração de deploy**, e num ambiente onde ele não esteja configurado as
  quatro rotas respondem **`503`** dizendo isso — em vez de `502` sem causa.

## Quem autoriza a leitura

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

> **O default é restritivo.** Entity **sem** `_conf.accessControl.read` declarado, ou com a lista
> vazia, **não é legível** — o persistence-q recusa. Autorizar leitura é ato explícito: declarar o
> papel na política da entity, o que vale de imediato, sem republicar modelo nem reiniciar serviço.

`/session/aggregate` e `/session/history` **mantêm** o portão de capacidade porque batem no
persistence-crs, que valida **tenant**, não leitura por papel — sem o portão, estado e histórico de
qualquer agregado ficariam ao alcance de qualquer portador de sessão daquele tenant.

## Limitar os campos que um papel lê — `readProjection`

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

**Três colunas nunca se recortam:** `id`, `aggregateid` e `status`. Sem elas a tela não seleciona
registro nem sabe que transições cabem, e cortá-las não protegeria dado pessoal nenhum — são
identificador e estado.

> **"Nunca se recortam" não é "sempre aparecem".** O recorte **preserva** essas três quando elas estão
> na resposta; ele não as injeta onde não estavam. As rotas carregam formas diferentes e é esperado:
> `query` devolve a projeção, com `id` **e** `aggregateid`; `aggregate` devolve o estado do write model,
> onde a chave é `id`; e no `history` o `eventData` traz só o que aquele comando carregava — um comando
> de criação não tem `id` nenhum. Medido nas três portas em 2026-09-13.

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

> **A plataforma não valida a declaração, e o BFF valida.** O forger aceita com `201` papel
> inexistente, coluna com nome errado e lista vazia: ele confere `aggregate`, `org/project/tenant` e
> os placeholders, e carrega o resto sem olhar. Então **declaração inválida faz a leitura ser recusada
> com `500`**, nomeando o que está errado — não recortada pela metade. Entregar tela que parece
> funcionar sobre uma regra de acesso que não se sustenta é pior que recusar.
>
> **Nunca prefixe a chave com `_`.** A publicação do `.model.json` remove chaves `_`-prefixadas em
> qualquer profundidade, **em silêncio** — a declaração sumiria sem aviso nenhum.

> ### A chave é declarada no esquema do forger
>
> `readProjection` é propriedade **declarada** do agregado no esquema com que o forger valida a
> publicação do `.model.json` — lido pelo `composer`, dono do forger, em 2026-09-25: a publicação a
> reconhece, e não depende mais de o forger tolerar chave desconhecida. **O BFF a lê** — é ela que faz o
> recorte desta seção. **Quem for mexer naquele caminho
> precisa saber que ela carrega política de acesso a dado pessoal.** A conferência do conteúdo — papel,
> coluna, lista vazia — continua sendo do BFF, que recusa a leitura com `500` quando a declaração não se
> sustenta.

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

## Atributo calculado pelo br — `computed`

Um comando às vezes tem atributo que **o processor de regra de negócio preenche** — o `username` tirado
do token, o nome copiado de outro agregado, a marca de "matrícula corrente". O usuário não tem o que
digitar ali, mas a chave **precisa chegar ao processor presente**: a mescla da resposta dele é por
whitelist, e ele só substitui chave que já veio no comando — chave nova é descartada em silêncio
([br-service — regra de negócio](../br-service/contextos.md)). E o BFF não repassa `""` nem `null`.

`computed` resolve isso de ponta a ponta: o modelo **declara** o atributo como calculado, a tela **não o
desenha**, e o BFF o **envia com um marcador**, que o processor sobrescreve.

**Onde se declara:** no `.model.json`, **ao lado dos comandos do agregado** — o mesmo lugar do
`readProjection` —, por comando. Nome solto é atributo de `data.attribute`; `"<vo>.<campo>"` é campo de
um value object declarado como grupo.

```jsonc
"cadastro.aluno": {
  "command": {
    "criar": {
      "data": {
        "attribute": {
          "nome":     { "type": "String", "length": 120, "nullable": false },
          "username": { "type": "String", "length": 60,  "nullable": false }
        },
        "valueObject": { "multiple": { "matriculas": {
          "planid":    { "type": "String",  "length": 36, "nullable": false },
          "iscurrent": { "type": "Boolean", "nullable": false },
          "planname":  { "type": "String",  "length": 80, "nullable": true }
        } } }
      },
      "br": { "route": "<org>/cadastro/cadastro/aluno/criar" }
    }
  },
  "computed": { "criar": ["username", "matriculas.iscurrent", "matriculas.planname"] }
}
```

- **O atributo continua declarado no comando.** `computed` não cria atributo: marca um que existe. E
  precisa existir — atributo que o comando não declara é removido antes de chegar ao processor.
- **Nunca prefixe a chave com `_`.** A publicação remove chaves `_`-prefixadas em silêncio.

**O que muda em cada ponta:**

| Onde | Efeito |
|---|---|
| `GET /session/capabilities` | o atributo vem com `role: "computed"` — em `attributes` e em `valueObjects[].fields` |
| miolo genérico | não desenha o campo; um value object em que **todo** campo é calculado não aparece |
| `POST /session/command` | o BFF envia o campo **sempre**, com um marcador, **ignorando** o valor que o consumidor tenha mandado. Em value object, o marcador vai em **cada item enviado** — value object ausente continua ausente, e lista vazia continua vazia |

**Os marcadores** — sempre um valor **explícito e não nulo**, um por tipo. Só o de texto muda conforme o
campo aceite nulo ou não. Todos passam pela validação que o persistence-crs faz antes do processor — ela
cobra `nullable` e o `length` de `String`, e normaliza as datas:

| Tipo | Marcador |
|---|---|
| `String` · `Text` **obrigatório** | `"computed"` — cortado ao `length` quando o campo é menor que isso |
| `String` · `Text` com **`nullable: true`** | `""` |
| `Integer` · `Long` | `0` |
| `Boolean` | `false` |
| `Date` | `"1970-01-01"` |
| `Timestamp` | `"1970-01-01T00:00:00Z"` |
| `Json` | `{}` |

**Por que o marcador nunca é `null`:** porque, no motor anterior, `null` explícito no comando **quebrava
a leitura**. Medido por um sistema consumidor em 2026-09-25, direto no persistence-crs:

| Motor anterior — forma enviada num campo `nullable` | Criação | Transição |
|---|---|---|
| chave **ausente** | materializa; write model `null`, projeção `""` | os dois lados mantêm o valor anterior |
| **`null`** explícito | campo numérico: **a projeção não materializa a linha** — o agregado existe e a consulta não o acha. Campo de texto: materializa com a **palavra `"null"`** na projeção | write model grava `NULL`, **projeção mantém o valor anterior** |
| `""` ou `"computed"` explícito | materializa, os dois lados iguais | os dois lados gravam o valor |

No motor anterior, `""` numa `Date`/`Timestamp` também respondia `200` e a gravação da projeção falhava.

**O motor foi consertado** ([persistence-crs — `null` explícito, chave omitida e o que o processor
devolve](../persistence-crs/endpoints/comando.md)): `null` explícito grava `NULL` nos dois lados, e `""`
numa data vira `null` no opcional e `400` no obrigatório. **O conserto está no ar em
`/v3/persistence/t/`; em `/v3/persistence/`, ainda não** (2026-09-25). Enquanto as duas instâncias não
rodarem o mesmo motor, só o valor explícito não nulo grava igual nas duas — por isso o marcador não é
`null`, e o das datas é o epoch, também no campo opcional. Entre o vazio e `"computed"`, o
opcional fica com o **vazio**: o bloco de campos que o processor deixa de preencher de propósito — os
dados de um empréstimo numa saída de estoque que não é empréstimo — termina vazio, e não com um nome
`"computed"` que a tela mostraria como dado. O obrigatório fica com `"computed"`, porque ali o processor
**tem** de preencher, e se esquecer o vazamento precisa ser visível.

O campo **dentro** de value object precisa do marcador tanto quanto o de topo: o persistence-crs cobra o
`nullable` dos campos internos **antes** do processor, e um item sem o campo obrigatório é recusado sem
a regra rodar.

**Do lado do processor:** em todo campo declarado `computed`, **devolver um valor, nunca `null`** — no
obrigatório, o valor calculado; no opcional que não se aplica, o vazio do tipo (`""`, `0`, `false`).
E **não validar o valor que chega** num campo `computed`: ele é sempre o marcador. Uma regra como
"maior que zero" sobre o valor de entrada recusa o marcador `0` — e com ele o próprio comando que
deveria calcular o campo.
Para campo de value object, a mescla é por chave de topo — devolve-se a lista inteira, com o campo
calculado em cada item:

```
função(data, authToken):
    usuario = decodifica(authToken)
    plano   = lê o plano de cada data.matriculas[i].planid
    retorna {
        ...data,
        username:   usuario.username,
        matriculas: data.matriculas.map(m => { ...m,
                        iscurrent: (m é a matrícula vigente),
                        planname:  plano(m).nome })
    }
```

> ### ⚠️ O marcador é gravado se o processor não o sobrescrever
>
> Se o processor **não devolver** a chave, ou devolvê-la `null`, o valor que fica é o **marcador** — o
> motor ignora o `null` na mescla e não revalida nada depois dela. Nada acusa: o comando responde `200`
> e o evento guarda `"computed"`, `0` ou `false` como se fosse dado. Quem escreve o processor é quem
> impede isso, devolvendo o campo em **todo** caminho que não lance erro.
>
> **Devolver `null` não limpa campo nenhum** — nem aqui, nem fora do `computed`, e nem no motor
> consertado: o `null` que o **processor devolve** é ignorado na mescla, e "apagar" e "não mexer" dão no
> mesmo ([br-service — o que devolver](../br-service/contextos.md)). É outra regra que a do `null` enviado
> no comando ([persistence-crs](../persistence-crs/endpoints/comando.md)).
> Para limpar, devolva o vazio do tipo.
>
> É o limite desta convenção, e é por isso que ela ainda não é a forma definitiva: a saída sem marcador
> é o próprio motor aceitar do processor qualquer atributo declarado no modelo — decisão pendente.

**Declaração inválida recusa o comando com `500`**, nomeando o que está errado — não envia marcador pela
metade. É inválido: chave que não é objeto `{ "<comando>": ["<atributo>"] }`; comando que o agregado não
declara; lista vazia; atributo ou campo de value object que o comando não declara; `status`; o
`whenAttribute` de um evento (o carimbo do servidor); e **comando sem `br.route`** — sem processor,
ninguém sobrescreveria o marcador, e ele seria gravado.

> ### A chave é declarada no esquema do forger
>
> `computed` é propriedade **declarada** do agregado no esquema do forger, na forma que o BFF lê
> ([forger — model](../forger/endpoints/model.md)). A publicação já recusa com `400` comando inexistente
> e atributo sem ponto fora de `data.attribute`. **O resto é conferido pelo BFF**, no comando: o nome
> com ponto (campo de value object), `status`, o `whenAttribute` e a falta de `br.route` — a recusa aí é
> o `500` descrito acima.

## `predicates` é a forma do persistence-q — não há dialeto do BFF

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
- **`predicates` é objeto. Array é recusado com `400`** e mensagem dizendo a forma esperada. Uma
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

> **Duas travas no autocadastro — uma nos dois lados, outra só no BFF.**
> 1. **O papel é validado contra o cardápio público** (`GET /ua/roles`, que lista só `ispublic=true`).
>    O orgid também passou a recusar papel não público com `403`
>    ([orgid — público](../orgid/endpoints/publico.md)); o BFF confere antes e mantém a checagem como
>    defesa em profundidade. Fora do cardápio → `403`.
> 2. **A conta nasce sempre `PENDING` — e esta trava é só do BFF.** O orgid ainda aceita
>    `account.status: ACTIVE` no corpo, e a chave `from` na raiz ativa a conta com qualquer valor; o BFF
>    força `PENDING` e não repassa `from`, de modo que só o fluxo de hash por e-mail ativa a conta.
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
| `400` | falta parâmetro (ex.: `username`/`password`, `tenant`), ou campo obrigatório do comando ausente · **valor fora do tipo declarado** no comando (`Boolean` que não é true/false, `Integer`/`Long` que não é inteiro) |
| `401` | sem sessão / sessão inválida / credencial inválida — ver `reason` abaixo |
| `403` | comando não autorizado ao papel · papel fora do cardápio público no autocadastro · papel que não é do usuário na org (`active-role`) |
| `404` | tenant não pertence ao usuário / miolo não registrado / **modelo do tenant ausente do cache** (removido ou nunca publicado — ele **não expira**) |
| `409` | autocadastro: papel inexistente — nada foi criado (o `204` do orgid, traduzido) |
| `500` | configuração ausente no servidor (ex.: o path do endpoint de cache não configurado) · **`readProjection` inválido no modelo do tenant** — a leitura é recusada em vez de recortada pela metade · **`computed` inválido no modelo do tenant** — o comando é recusado em vez de gravar marcador |
| `413` | arquivo acima do teto do filer (3 MiB) em `POST /session/files/upload` |
| `502` | falha ao falar com um serviço da plataforma |
| `503` | serviço de arquivos (filer) não configurado neste ambiente |

### `401` diz **por que** não há sessão

O corpo do `401` de sessão traz `reason`, para o consumidor distinguir quatro situações que, sem ele,
chegariam iguais:

| `reason` | Significado | O que o consumidor deve fazer |
|---|---|---|
| `no-session` | não há sessão — ninguém entrou ainda | pedir login, normalmente |
| `expired` | o prazo da sessão acabou | pedir login de novo |
| `evicted` | a sessão **desapareceu do servidor antes de expirar** | pedir login **e preservar o destino**: o portador não fez nada errado |
| `unknown` | não foi possível determinar | tratar como `expired` |

**O corpo traz também `expiresAt`** (epoch em segundos) **quando o BFF sabe o instante em que a sessão
venceria** — ele vem de um cookie irmão, assinado, que guarda só essa data. Serve para o consumidor
distinguir *"venceu agora"* de *"venceu há muito"*, e é o que dá substância ao `evicted`: sessão que
sumiu **antes** desse instante não expirou, foi despejada. Ausente quando não há como saber.

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
- [ ] Em `/session/files/*`: `entityId` da chave = **`aggregateid`**; prefixo em `delete` é **lote**; e
      o anexo é protegido **por papel na entity**, nunca por titular — ver o aviso da seção.
- [ ] Atributo que o processor preenche: declare-o em `computed`, e faça o processor **sempre**
      devolvê-lo não nulo — senão o marcador é gravado.

---
