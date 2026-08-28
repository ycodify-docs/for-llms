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

**Como confirmar que é o BFF e está no ar:**

```
GET https://<endereço-do-bff>/health   →   200  {"ok":true,"service":"yc-app-bff"}
```

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

O BFF lê o **modelo publicado** do tenant no **[cache](../cache/README.md)** — **não** no forger (o
forger é quem **grava** ali ao publicar o `.model.json`) — e cruza com os **papéis org-scoped** do token
→ devolve **só** os comandos autorizados. Só para tenant que **consta no token**; senão `404`.

> ⚠️ **A capacidade depende do modelo estar VIVO no cache.** O modelo publicado tem **TTL**: ao expirar,
> o cache não devolve mais (204) e a capacidade do bounded context **some** — `GET /session/capabilities`
> deixa de listar os comandos daquele tenant, **mesmo com o sistema provisionado e o dataschema
> `RUNNING`**. Não é falha do BFF: é o modelo que saiu do cache. Remediação: **republicar** o
> `.model.json` (`POST forger .../tenant/<id>/model`, sobrescrita idempotente). Vale para **qualquer**
> consumidor — casca ou cliente sem UI.
>
> Quem define esse TTL é **o forger, no momento da publicação** (é ele que grava a chave com prazo); o
> cache apenas honra o que recebeu. O prazo é **configuração de deploy** do forger — em ambiente de
> desenvolvimento, um sistema que fica dias sem republicar perde a capacidade sozinho.

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
- `/session/aggregate` existe porque a **projeção é assíncrona**: para carregar o estado autoritativo de
  um agregado (ex.: preencher um form de transição) não se deve ler o read model.

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
| Health | `GET /health` | `{ "ok": true, "service": "yc-app-bff" }` |

Sem autenticação e sem estado — serve ao healthcheck do container e ao balanceador. Não é caminho de
consumo: um cliente não precisa dele.

## Erros

| HTTP | Quando |
|---|---|
| `400` | falta parâmetro (ex.: `username`/`password`, `tenant`), ou campo obrigatório do comando ausente |
| `401` | sem sessão / sessão inválida / credencial inválida |
| `403` | comando não autorizado ao papel · papel fora do cardápio público no autocadastro · papel que não é do usuário na org (`active-role`) |
| `404` | tenant não pertence ao usuário / miolo não registrado / **modelo do tenant ausente ou expirado no cache** |
| `409` | autocadastro: papel inexistente — nada foi criado (o `204` do orgid, traduzido) |
| `500` | configuração ausente no servidor (ex.: o path do endpoint de cache não configurado) |
| `502` | falha ao falar com um serviço da plataforma |

## Checklist do agente

- [ ] O browser fala **só** com o BFF — nunca com a plataforma direto.
- [ ] O token **nunca** vai no corpo — só em **cookie httpOnly**.
- [ ] `Authorization`, `X-Tenant-Id` **e `X-Forger-Credential`** são compostos **no BFF**
      (ver [seguranca](../shell/seguranca.md)).
- [ ] No autocadastro, ofereça só o que `GET /ua/roles` devolve — o servidor recusa o resto.
- [ ] Endereços de serviço = **config de deploy**, nunca hardcode nem em doc pública.
