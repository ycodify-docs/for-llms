# CHANGELOG — documentação pública (for-llms)

> Histórico de revisões desta documentação. Datas em formato `AAAA-MM-DD`.

## 1.17 — 2026-08-08

- **Autocadastro de usuário de aplicação (self-service) documentado.** Nova seção em `orgid/README.md`
  ("Autocadastro de usuário de aplicação (modo da plataforma)"): padrão em **2 etapas** — conta `/ua` + papel via
  `POST /open/ua/account-role` (público) → **registro de domínio** (agregado no persistence-crs) com o `username`
  **como identidade / chave de unicidade** (1 conta = no máx. 1 registro). **Qual papel = decisão de design do
  sistema** (default fixo OU cardápio `ispublic` via `GET /ua/open/role/owner/{roleOwner}`); o papel precisa
  **pré-existir** (senão `204` = nada criado). Recuperação **CQRS-ES** antes de transicionar. **Autorização "só o
  próprio registro" NÃO vem do token** — é regra de negócio (ex.: processador br-service).
- **Descoberta**: ponteiro de topo em `06-autenticacao.md` + linha no índice orgid do `llms.txt` → a seção.

## 1.16 — 2026-08-02

- **orgid: `POST /open/ua/account-role` documentado por completo — e o `204` dele NÃO é sucesso.** O
  endpoint público cria a conta externa **e** o vínculo com um papel **já existente** em **uma única
  transação**. Papel (`name`+`owner`) inexistente → **`204` sem corpo** e **nada** é criado (nem a
  conta): a transação é revertida. Só `200` confirma o registro.
- **`role.owner` é o `name` da organização** dona do papel (antes descrito apenas como "espaço de
  nomes"). E `role.label`/`role.ispublic`/`role.status`/`role.id` no corpo são **aceitos e ignorados** —
  a doc os chamava de "metadados do papel", sugerindo um efeito que não existe: o papel vem do cadastro.
- **Validações do registro agora na doc.** `username`: mín. 3 caracteres, só letras e dígitos, `yc`
  reservado, normalizado para minúsculas e **único em toda a plataforma** (não por organização) —
  duplicado → **`409`**, código antes ausente do catálogo. `password`: ao menos 1 maiúscula, 1 minúscula
  e 1 dígito. `email`: formato `algo@algo`.
- **Estado inicial da conta.** `account.status` aceita só `PENDING` ou `ACTIVE`; qualquer outro valor
  vira `PENDING` **em silêncio**. `ACTIVE` cria a conta já ativa; `PENDING` exige o fluxo de hash.
- **Chave `from` na raiz do corpo: só a presença importa.** Existindo a chave — com qualquer valor,
  inclusive `false`/`null` — a conta nasce **ativa** e `account.status` é sobreposto. Preferir
  `account.status`, que deixa a intenção legível.
- **Corpo fora da forma → `510`, não `400`:** propriedade **desconhecida** dentro de `account` ou de
  `role`, ou corpo sem `account`/`role`.
- Atualizados: `orgid/endpoints/publico.md` (seção reescrita), `orgid/erros.md` (`409`, nota do `204`
  enganoso, categoria "Conflito"), `orgid/exemplos.md` (exemplo #8), `orgid/openapi.yaml`
  (`UaRegistroAccountRole`, resposta `Conflict`, path `/open/ua/account-role`), `llms-full.txt`.

## 1.15 — 2026-08-02

- **CORREÇÃO (persistence-q): posição dos controles de consulta e forma do `_sorting`.** A doc dizia que
  `_paging`/`_sorting`/`_count`/`_connective`/`_cache` iam **dentro** do objeto do rótulo — **errado**. O
  serviço lê esses controles no **nível raiz** do critério (irmãos do rótulo); postos dentro do rótulo são
  **ignorados em silêncio** (aplica default). Além disso `_sorting` é **indexado por posição**
  (`{ "0": { "_orderBy", "_order" } }`, até `"1"`/`"2"`), não plano. Confirmado por código
  (`QueryServiceImpl.readByCriteria` + `ReadStatementTransformation`) + teste git-tracked
  (`TestReadStatementJSON`).
- **Conectivo `OR` documentado.** `_connective` (nível raiz) = `AND` (padrão) ou `OR` (maiúsculas), global
  ao critério; não mistura AND/OR num mesmo critério; sub-condições de atributo composto são sempre AND.
  Exemplos de `OR` (igualdade e operadores) adicionados.
- Atualizados: `persistence-q/query-controls.md` (reescrito), `endpoints/consulta.md`, `exemplos.md`
  (paging/sorting/OR/count), `README.md`, `openapi.yaml` (`Criterio`: controles no nível raiz, removido
  `maxProperties:1`), `llms-full.txt`.
- **Nota:** os controles de associação (`_associations`/`_populating`/`_level`/`_as`) permanecem **dentro**
  do rótulo e **não têm consumidor** no código de consulta atual (provável tratamento na camada de
  aplicação/BFF); mantidos na doc como estão, a confirmar.

## 1.14 — 2026-08-01

- **URL de callback do processador: endereço e path por variável de ambiente (br-service).** O
  processador obtém o destino de rede em `PLATFORM_ENDPOINT_PERSISTENCE_C` /
  `PLATFORM_ENDPOINT_PERSISTENCE_Q` e o path do endpoint interno em
  `PLATFORM_ENDPOINT_PERSISTENCE_Q_UNSEC_PATH` — injetadas no ambiente do serviço e disponíveis a
  **qualquer** processador, em qualquer nível de pasta, sem passar pelo payload. Variável ausente →
  `400` nomeando a variável faltante. Antes a doc dizia que o path vinha "de variável de ambiente" sem
  nomear nenhuma, o que levava o leitor a supor que a URL interna era a raiz do endereço-base.
- **Duas superfícies espelhadas, e a escolha é ditada pelo tipo de hook.** Cada operação existe na
  superfície **pública** (via gateway, `Authorization` + `X-Tenant-Id`) e na **interna de cluster**
  (prefixo próprio, só `X-Tenant-Id`, negada pelo gateway). Hook síncrono tem JWT e pode as duas;
  coordenação e projeção não têm JWT e só podem a interna. Antes o texto dizia apenas "endpoint interno,
  path via variável de ambiente" sem nomear variável nem contrastar com a superfície pública — daí a
  suposição frequente de que bastava apontar para o endereço-base.
- **Contrato de resposta `400`: são duas formas, não uma.** Falha de **validação do corpo** devolve
  `{erro}`; **rota inexistente** e **exceção no processador** devolvem `{status, mensagem, tipo}`.
- **Ciclo de vida: o processador pode receber até três argumentos.** Com `data`, `authToken` e
  `tenantIds` no corpo, é invocado como `(data, authToken, tenantIds)`; sem `tenantIds`, como
  `(data, authToken)`; só com `data`, como `(data)`; sem `data`, recebe o corpo inteiro. `authToken` e
  `tenantIds` só chegam na raiz do corpo no hook síncrono; `tenantIds` traz o `tenantId.forReadModel` do
  modelo do agregado.
- **JWT em hook assíncrono: oportunista, não garantido.** Parte dos fluxos de coordenação propaga o token
  do usuário de origem em `data._meta.authToken` (para que a escrita de comando use a credencial de quem
  originou o evento, não uma credencial de serviço fixa); outros fluxos não enviam token algum. Nunca
  chega como argumento. Um processador assíncrono precisa funcionar sem token.
- **Forma canônica da rota: o serviço não a valida.** A unicidade global depende da disciplina de quem
  publica; sem o prefixo, duas organizações acabam apontando para o mesmo processador.
  Atualizados: `br-service/README.md`, `CHANGELOG.md`.

## 1.13 — 2026-07-31

- **CRUD de entidade convencional (`POST /e`) documentado para o cliente.** Antes, `/e` só constava como
  "escrita de projeção (serviço)". Agora há doc cliente para CRUD direto de **entidade convencional**
  (tabela não-agregado, identificada por `id`/PK), distinta do caminho de **agregado** (`/a`, comando/
  evento). Aviso: não usar `/e` em entity que é projeção de agregado event-sourced (desalinha a projeção).
  Novo: `persistence-crs/endpoints/entidade.md`. Atualizados: `persistence-crs/endpoints/projecao.md`
  (cross-link + enquadramento de uso interno), `persistence-crs/README.md` (índice + papel),
  `persistence-crs/exemplos.md`, `persistence-crs/openapi.yaml` (`/e`: summary + `403`), `llms.txt`,
  `README.md` ("Por tarefa"), `llms-full.txt` (regenerado).

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
- Bundle (`build-bundle.sh`): allowlist passa a incluir `.yaml`.
- `dataschema.md`: `status` documentado como **rótulo administrativo editável** (sem lifecycle imposto
  nem gate para criar entity). O ciclo "MODELING→RUNNING→STOPPED" da proposta **não** existe no código
  (drift do mirror); `ACTIVE` é status de broker, não de dataschema.

## 1.1 — 2026-06-20

- Guardião dividido: `docs/CLAUDE.md` (sensível, não publicado) + `public/CLAUDE-extended.md` (público).
- `llms.txt` como índice primário; `llms-full.txt` como fallback (cabeçalho + ordem alto-valor-primeiro).
- Regras contexto-aware (teto de tamanho, fatos-chave no topo, ordenação) no guardião.
- **Fronteira de publicação:** `publish/build-bundle.sh` monta o bundle (allowlist `for-llms`, exceto
  `CLAUDE*`), com guardas (sem CLAUDE, leak-scan, links). A app serve só `publish/dist/`.

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
