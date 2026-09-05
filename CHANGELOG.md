# CHANGELOG — documentação pública (for-llms)

> Histórico de revisões desta documentação. Datas em formato `AAAA-MM-DD`.

## 1.16 — 2026-09-05

- **Escrita que não acha o alvo responde `404`, não `204` — 31 operações do orgid.** A 1.15 corrigiu um
  caso (`PUT /ua/role`); a varredura mostrou que o padrão era idiomático no serviço. `204` é família 2xx —
  sucesso sem corpo —, então uma escrita que falhava era **indistinguível de uma que gravou**: os dois
  respondiam 2xx e vazio, `response.ok` era `true` nos dois, e a mensagem de erro era descartada pelo
  próprio protocolo, que proíbe corpo em `204`. Agora cada `404` traz **o efeito que não ocorreu**, não só
  o objeto que faltou: `"account not found: the password was not changed."`, `"O papel informado não
  existe: nada foi removido."` Os dois casos mais perigosos que isso encerra: `DELETE` de papel inexistente
  (num `DELETE`, `204` **é** o código canônico de sucesso — a leitura era a oposta da correta) e
  `replace-master-account`, em que trocar a conta administradora de uma organização virava no-op
  silencioso. **`204` continua em leitura**, incluindo os `/exists`, onde `200`/`204` é o contrato e não um
  erro. Atualizados `orgid/erros.md` (linha `404` nova + nota), `orgid/README.md` (ciclo de vida) e as
  seções de escrita de `ua-papel.md`, `ua-conta.md`, `ua-associacao.md`, `up-conta.md`,
  `up-organizacao.md`, `up-associacao.md` e `publico.md`.

- **⚠️ `/ua/open/role/owner/{owner}` é o único público do orgid fora do prefixo `/open/`.** Os segmentos
  estão invertidos, e por isso ele escapa de qualquer regra de autorização derivada de `/open/**` — no
  `OrgIdSecurityConfig` do orgid standalone ele **exige token apesar de documentado como público**. O
  `composer`, que é o que está implantado, libera os dois prefixos desde 2026-09-05. Registrado em
  `orgid/endpoints/publico.md` e `ua-papel.md`.

- **Duas correções de comportamento que a doc não trazia:** conta `/ua` nasce `PENDING` e **já loga** — o
  sign-in não checa `status`, e a ativação por hash serve para provar posse do e-mail, não como portão de
  acesso (`ua-conta.md`); e **registro não envia e-mail** — quem envia é o fluxo de hash e
  `/open/ua/account/e-mail/send`, contra provedor real com cota, o que desaconselha acioná-los em teste
  (`publico.md`).

## 1.15 — 2026-09-05

- **`PUT /ua/role` exige `id`, e a documentação não dizia.** O texto era "mesmos campos do POST" — e o
  POST **não tem `id`**. Quem seguia a doc ao pé da letra montava um corpo sem `id`, e o serviço
  respondia **`204`**: família 2xx, sucesso sem corpo. A mensagem "role not found." era descartada pelo
  próprio protocolo, e o chamador seguia achando que tinha configurado. Um cliente marcou dois papéis
  como públicos assim e perdeu tempo até descobrir que o valor no banco nunca mudara. A seção agora
  mostra o corpo com `id`, diz como descobri-lo (`GET /ua/role/by/name/{roleName}/owner/{roleOwner}`),
  e registra os códigos corretos: `400` sem `id`, `404` quando o papel não existe ou é de outro `owner`.
  O `204` foi corrigido no serviço na mesma data. Em `orgid/endpoints/ua-papel.md`.

## 1.14 — 2026-09-03

- **Correção de fato: não existe recuperação automática do modelo a partir do forger.** A documentação
  descrevia um *self-heal* — "se falta no cache, recupera do Forger e repõe" — que **não existe mais**: o
  serviço de implantação passou a ser o **único** publicador do modelo no cache, e o caminho de leitura
  **só lê**. Um miss é **parada** (`510`, "republique o modelo"), não um caminho lento que se resolve
  sozinho; e, como a entrada **não expira**, um miss significa "nunca publicado" ou "removido". Corrigido
  em `persistence-crs/README.md` (dono do fato) e `persistence-q/README.md`; consequência registrada em
  `03-fluxo-de-deploy.md` (a publicação é a única porta de entrada).
- **Regra nova: depois de um comando, espere antes de consultar.** O polling contra o serviço de consulta
  deve começar por um **atraso inicial em milissegundos** e reconsultar com tentativas limitadas — nunca
  disparar a primeira consulta junto com a resposta do comando. A razão passa a estar escrita: o
  mecanismo que persiste a **projeção** é **desvinculado** do que persiste o **agregado**, logo o `200`
  do comando **não afirma nada** sobre a projeção. Inclui a consequência que mais engana: uma falha na
  projeção **não aparece** na resposta do comando. Em `persistence-q/README.md`.
- **`_cache: "use"` marcado como não utilizável.** O controle está documentado, mas hoje erra nos dois
  caminhos: sem entrada, responde sem os registros **e a consulta nem executa**; com entrada, devolve o
  invólucro em vez do resultado. Enquanto não houver correção, a orientação é omitir `_cache` ou usar
  `"ignore"`. Em `persistence-q/query-controls.md`.
- **`envtype` é rótulo, não seletor.** O campo é obrigatório na criação do project, mas **não** escolhe
  banco, esquema, instância nem rota, e nenhum processamento o consulta — preenchê-lo com `PRODUCTION`
  não torna nada produtivo. Documentado como anotação administrativa, com ponteiro para a única forma
  de saber o destino real. Em `forger/endpoints/project.md`.
- **Nova seção: "descobrir para onde um tenant projeta".** O destino das projeções não está no modelo
  nem no `tenant-id`; vem do provisionamento. Documentado o percurso da cadeia
  `model → dataschema → database → dbconn` com os endpoints autenticados de cada passo, e o motivo:
  antes de qualquer ação com consequência sobre os dados de um tenant, é a cadeia que diz qual banco a
  ação alcança. Em `forger/README.md`.
- **Alterar `dbconn` não vale de imediato.** A conexão resolvida do tenant fica guardada em memória por
  um intervalo; até vencer, as requisições seguem usando endereço e credencial antigos — e uma falha de
  autenticação logo após a troca não significa que o novo valor está errado. Em
  `forger/endpoints/dbconn.md`.
- **Dois diagramas novos.** (1) *Como o `tenant-id` vira o banco de um cliente* — separa as duas
  resoluções que partem do mesmo `tenant-id`, a **conexão** (vem do forger) e o **modelo** (vem do
  cache), em `02-conceitos.md`. (2) *Qual banco de leitura, e quando isso é decidido* — mostra que o
  destino da projeção é resolvido **no consumo da fila**, a partir do `tenant-id` **da mensagem**, e
  portanto **depois** da resposta do comando, em `01-arquitetura.md` §Fase 3.

## 1.13 — 2026-09-02

- **`PUT` de entity: atualização parcial de verdade — ausente ≠ vazio.** A documentação já prometia
  *"campos omitidos são ignorados, não apagados"*, mas o serviço tratava propriedade **ausente** como
  pedido de reset ao default: um `PUT` que enviava só `accessControl` **apagava o `uniqueKey`** da
  entity, sem nada na resposta indicando a remoção. O mesmo mecanismo atingia `superEntity` (desfazia
  herança) e, no nível de atributo, derrubava `NOT NULL`/`unique` e apagava o comentário. Corrigido no
  serviço; o contrato agora é explícito:
  - propriedade **ausente** → ignorada (valor atual preservado);
  - propriedade com **valor vazio** (`[]`, `""`) → **limpa**;
  - `null` explícito conta como **ausente** (não é sentinela de limpeza);
  - `accessControl` é **substituído por inteiro**, não mesclado chave a chave (enviar só `read` reseta
    `write` para `["MASTER"]`);
  - `_conf` é **opcional** no `PUT` (segue obrigatório na criação).
- **`PUT` passou a responder com o que aplicou.** O `200` antes vinha **sem corpo**; agora traz
  `{entityName, valid, summary, totalChanges, applied[], skipped[]}`, com valor antigo e novo de cada
  propriedade alterada — uma alteração que remove constraint passa a dizê-lo.
- **Correção de fato:** `_conf.superEntity` e `_conf.superEntityStrategy` **são** atualizáveis; o BNF
  os listava como imutáveis.
- **Chave única: `attribute.unique` e `_conf.uniqueKey` são ortogonais e podem coexistir.** Documentado
  o que antes era conhecimento tácito: unicidade de um atributo isolado vs chave composta (2+ atributos).
  Combinar os dois é legítimo quando incidem sobre atributos **diferentes**; quando o **mesmo** atributo
  é `unique` e também está no `uniqueKey`, a composta é logicamente redundante (aceita, mas sem efeito
  prático além do custo). A ordem declarada em `uniqueKey` não é preservada.
- Atualizados: `forger/endpoints/entity.md` (novas seções "Chave única" e "Semântica do `PUT`: ausente
  vs vazio" + corpo da resposta), `forger/bnfs/update-entity-request.bnf`, `forger/openapi.yaml`
  (`EntityPartialUpdate`, `ChangeReport`, `ChangeEntry`).

## 1.12 — 2026-07-05

- **Contrato de identidade `id`: projeção (Long) vs agregado (UUID).** A projeção (read model) tem DOIS
  identificadores — `id` (**Long**, PK da linha) e `aggregateid` (**UUID**, o id do agregado;
  `projecao.aggregateid == aggregate.id`). Os endpoints persistence-crs (`/a/{bc}/{type}/{id}`:
  comando/transição, leitura de estado, `/history`) exigem o **UUID** (`aggregateid`); enviar a PK `id`
  (Long) → `510` "Invalid UUID string" (achado no E2E `biblioteca`). Atualizados:
  `persistence-crs/spec/model-format.md`, `.../erros.md`, `.../endpoints/agregado-leitura.md`,
  `persistence-q/README.md` + `exemplos.md` (exemplos corrigidos p/ `id`:Long + `aggregateid`:UUID),
  `02-conceitos.md`, `05-antipatterns.md`.

## 1.11 — 2026-06-27

- **orgid: novo campo obrigatório `client` na criação de organização.** `client` = **nome da empresa
  cliente** que contratou a plataforma (toda org deve sinalizar). Obrigatório em `POST /up/org` e no
  registro `POST /open/up/account` (`org.client`); **máx. 12 caracteres** (ausente/vazio ou >12 → `400`).
  Atualizados: `up-organizacao.md`, `publico.md`, `orgid/README` (conceito), `openapi.yaml`, `exemplos.md`.
- No `PUT /up/org`, `client` **não** é listado como editável (é definido na criação). *(Motivo interno,
  fora da doc: a alteração de `client` via PUT não persiste — defeito no serviço a corrigir.)*

## 1.10 — 2026-06-27

- **CORREÇÃO de fato (supersede 1.2/1.3): o `status` do dataschema É um gate operacional.** As versões
  1.2/1.3 documentaram `status` como "rótulo administrativo, sem gate" — **errado**. Verdade (confirmada
  pelo dono): há dois estados que **controlam** operação:
  - **`MODELING`** — esquema editável (criar/alterar `entity`); o modelo **não** é interpretado/executado
    (persistence-crs/persistence-q **não operam**).
  - **`RUNNING`** — esquema congelado (o **forger rejeita** alterar `entity`); persistence-crs/persistence-q
    **interpretam e executam** o modelo.
  - Para editar schema de sistema em operação: `RUNNING → MODELING → editar → RUNNING`.
- Arquivos atualizados: `forger/endpoints/dataschema.md` (gate substitui o parágrafo "sem gate"),
  `forger/endpoints/entity.md` (pré-condição `MODELING`), `persistence-crs/README.md` e
  `persistence-q/README.md` (pré-condição de runtime `RUNNING`), `03-fluxo-de-deploy.md` (passo 7
  "ativar"), `04-walkthrough.md` (passo 7b), `05-antipatterns.md` (gate nos dois sentidos).
- Origem: o gate vive nos serviços que interpretam o modelo (que ignoram `MODELING`); a recusa no forger
  é parte do contrato documentado.
- **Limites de nome atualizados:** dataschema **≤16** (era 12), entity/atributo/associação **≤24** (era
  12), superentidade **≤24** (era 64). Propagado a `forger/README`, `dataschema.md`, `entity.md`,
  `05-antipatterns.md`.
- **Rename:** `autenticacao.md → 06-autenticacao.md` (numeração); refs repontadas (`llms.txt`,
  `04-walkthrough`, `auth/README`, `auth/exemplos`, `orgid/README`).

## 1.9 — 2026-06-26

- **orgid · reescrita dos endpoints (por domínio, detalhe por endpoint).** A doc anterior misturava os
  domínios `/up` e `/ua` num mesmo arquivo e listava só a URL. Substituída por **arquivos por domínio**
  (`up-organizacao`, `up-conta`, `up-associacao`, `up-contrato`, `ua-conta`, `ua-papel`, `publico`),
  cada endpoint numa **subseção própria** com: **método HTTP**, **path-variables** (com significado),
  **cabeçalhos** (`Authorization`; sem `X-Tenant-Id`), **papéis** exigidos, **corpo** de POST/PUT
  (tabela de campos) e **resposta/status**. Índice do `orgid/README.md` e `llms.txt` repontados.
- Removidos os 6 arquivos antigos (`organizacao`, `conta`, `papel`, `associacao`, `contrato`,
  `registro-publico`); referências externas (README raiz, auth) repontadas para `publico.md`.
- Sanitização: omitidos campos que revelam nuvem/credenciais e o campo de bypass por segredo no envio
  de e-mail; identificador de pagamento mantido como termo genérico.
- **`/ua`:** a associação **conta-papel** (`/ua/account-role`) saiu de `ua-papel.md` para arquivo próprio
  `ua-associacao.md` (espelha o `up-associacao.md`); `ua-papel.md` fica só com **papel**.

## 1.8 — 2026-06-23

- **Novo serviço documentado: auth (emissão de token / login)** — `POST /up/sign-in`, `POST /ua/sign-in`,
  `GET /up/sign-in/renew`. Guia + endpoints + erros + exemplos + OpenAPI 3.1. auth **emite** o token de
  acesso (com identidade: usuário, papéis, organizações, tenants); os demais serviços **validam** (CP-12).
- Posicionado como **camada de identidade, par do orgid** (auth emite, orgid administra): 00-plataforma
  (8 serviços), 01-arquitetura, coordenação (CP-12), llms.txt, README.
- **autenticacao.md reconciliado:** a emissão do token agora aponta para o **auth** (não mais "fora de escopo").
- Sanitização: lib de token/criptografia, prefixos ofuscados, headers de rastreamento, padrão de sessão,
  hosts e segredos de token não documentados.

## 1.7 — 2026-06-23

- **forger (código): normalização de `schema.forWriteModel.name` na publicação do `.model.json`.** O
  armazém de escrita é universal; o forger impõe o valor canônico — se o JSON enviado não contiver um
  dos dois valores válidos, é convertido automaticamente (e, se ausente, definido). Doc atualizada
  (model-format, forger/model) — sem expor os valores reais (representados como `wdb.client`).

## 1.6 — 2026-06-23

- **Novo serviço documentado: filer (arquivos)** — upload/download/listar/remover de arquivos anexos a
  agregados. Guia + 4 endpoints + erros + exemplos + OpenAPI 3.1. Auth `Authorization`+`X-Tenant-Id`;
  RBAC por entidade (leitura/escrita). Chave `{org}-{project}-{entity}-{entityId}-{attribute}.{ext}`
  amarra o arquivo ao agregado; um agregado pode ter vários arquivos (por atributo).
- Posicionado como **serviço transversal**: 00-plataforma (7 serviços), 01-arquitetura, coordenação
  (CP-11), llms.txt, README.
- Sanitização: armazenamento de objetos/cache/infra abstraídos; `code` (chave de função) e detalhes
  de nuvem/rede não documentados.

## 1.5 — 2026-06-23

- **Novo serviço documentado: orgid (IAM)** — identidade e acesso (organizações, contas, papéis/RBAC,
  contratos). Guia + endpoints (organização, conta, papel, associação conta-papel-org, contrato,
  registro público `/open/*`) + erros + exemplos + OpenAPI 3.1. Dois domínios (`/up` plataforma, `/ua`
  externo) + público por design (`/open`). Valida token (não emite).
- Posicionado como **pré-requisito de identidade** (antes do forger): 00-plataforma (6 serviços),
  01-arquitetura (Fase 0), 03-fluxo-de-deploy (passo 0), coordenação (CP-8/CP-9/CP-10), llms.txt, README.
- Excluídos (não-públicos): endpoints internos de uso entre serviços (prefixos ofuscados), de teste e
  de observabilidade/administração.

## 1.4 — 2026-06-20

- **Convenção `_`-prefixo (metadados):** chaves de topo do `.model.json` começando com `_` (ex.:
  `_comment`, `_meta`, `_schemaVersion`) são metadados não-semânticos — o forger as **remove** (validação
  + JSON) e o grid de interpretação ignora. Use `_comment` para comentário livre; `comment` sem prefixo
  no topo quebra o forger (confundido com a chave do bounded context).
- Exemplos: `comment`/`$comment` de topo → `_comment` nos 6 modelos.
- Schema/`model-format.md`: reconhecem chaves de topo `_`-prefixadas; nota da exceção `attribute.comment`
  (semântico → comentário de coluna; permanece sem `_`).
- forger: o serviço de cache de modelo remove **recursivamente** toda chave `_`-prefixada (qualquer nível)
  (antes só `_schemaVersion`/`_meta`, top-level). Risco avaliado: nenhum consumidor lê `_x` do modelo. Requer rebuild/deploy.

## 1.3 — 2026-06-20

- **`persistence-q/query-controls.md`** (NOVO, verificado no código): operadores de predicado
  (`eq/neq/gt/gte/lt/lte/like/ilike/in`), `_connective` (AND/OR), `_paging`
  (`_maxRegisters`/`_firstRegister`), `_sorting` (`_orderBy`/`_order`), `_count`, `_cache`
  (`_behavior`: use/evict/ignore), e controles de associação (`_associations`/`_populating`/`_level`/`_as`).
  Era o maior gap real do corpus. Indexado em `consulta.md`, `README`, `llms.txt`.
- Itens da proposta (DEMANDA 2026-06-20) **não** portados por não se confirmarem no código: ciclo de
  estado do dataschema (MODELING→RUNNING→STOPPED), prereq `dataschema=MODELING` para entity, e
  "type-mapping enum" no forger (forger usa os 8 tipos direto; Decimal→Long/uuid→String(36) são
  convenções de autoria, não regra do serviço).

## 1.2 — 2026-06-20

- **OpenAPI** (3.1) por serviço: `forger/openapi.yaml`, `persistence-crs/openapi.yaml`,
  `persistence-q/openapi.yaml`, `br-service/openapi.yaml` (tech-agnóstico; só `Authorization`/`X-Tenant-Id`).
- **`05-antipatterns.md`**: o que mais quebra, por serviço (sintoma → causa → correção).
- `model-format.md`: identificação na projeção (linha por `aggregateid`; `status` = estado atual).
- Fronteira de publicação: a allowlist passa a incluir `.yaml`.
- `dataschema.md`: `status` documentado como **rótulo administrativo editável** (sem lifecycle imposto
  nem gate para criar entity). O ciclo "MODELING→RUNNING→STOPPED" da proposta **não** existe no código
  (drift do mirror); `ACTIVE` é status de broker, não de dataschema.

## 1.1 — 2026-06-20

- Guardião dividido: `docs/CLAUDE.md` (sensível, não publicado) + `public/CLAUDE-extended.md` (público).
- `llms.txt` como índice primário; `llms-full.txt` como fallback (cabeçalho + ordem alto-valor-primeiro).
- Regras contexto-aware (teto de tamanho, fatos-chave no topo, ordenação) no guardião.
- **Fronteira de publicação:** definida a allowlist do que é publicado e a denylist do que nunca sai,
  com as guardas de verificação antes de disponibilizar.

## 1.0 — 2026-06-20

Primeira versão consolidada da documentação pública para agentes de IA.

- Documentados os cinco serviços (forger, persistence-crs, es-n, persistence-q, br-service): guia,
  endpoints, erros e exemplos, com ciclo de vida de cada requisição.
- Documentos transversais: conceitos, composição, arquitetura ponta a ponta, fluxo de deploy,
  walkthrough único, autenticação/tenant, coordenação (CP-1…CP-7).
- **Gramática formal do `.model.json`** ([model-format.md](persistence-crs/spec/model-format.md)) +
  **JSON Schema** ([model.schema.json](persistence-crs/spec/model.schema.json)); os 6 modelos de
  exemplo validam contra o schema.
- Gramáticas **BNF** dos payloads do forger ([forger/bnfs/](forger/bnfs/README.md)).
- Índices: [`llms.txt`](llms.txt) (navegável) e [`llms-full.txt`](llms-full.txt) (corpus num arquivo).
- Sanitização **tech-agnóstica total**: sem nomes de produto/stack/versão, classes, filas, hosts,
  cabeçalhos internos, endpoints sem autenticação, UUIDs reais ou nomes de cliente.

> Para regenerar `llms-full.txt` após editar qualquer `.md`, concatene os arquivos na ordem do
> `llms.txt`.
