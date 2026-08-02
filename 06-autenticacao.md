# Autenticação e identificação de tenant

> Como formar uma requisição válida: a **URL base** (via API Gateway, prefixo `/v3/<svc>`) + o cabeçalho
> **`X-Forger-Credential`** (credencial do gateway) + o cabeçalho `Authorization` + o cabeçalho de tenant
> `X-Tenant-Id`. Pré-requisitos: [conceitos](02-conceitos.md).

## URL base — via **API Gateway** (`/v3/<svc>`)

O acesso aos serviços é por um **API Gateway**: **um host** (a base) e o serviço é escolhido pelo **prefixo do
path `/v3/<svc>`**. A base é env-specific:

| Ambiente | Base (host do gateway) |
|---|---|
| produção | `https://api.ycodify.com` |
| dev local | `http://localhost:8080` |

Prefixo por serviço — o gateway **remove** o prefixo antes de encaminhar, então os paths documentados em cada
serviço (`/org/...`, `/a/{bc}/{type}`, `/up/sign-in`, …) são os **downstream**: anexe-os **após** o prefixo.

| Serviço | Prefixo | Exemplo (prod) |
|---|---|---|
| forger | `/v3/forger` | `https://api.ycodify.com/v3/forger/org/{org}/dbconn` |
| auth | `/v3/auth` | `https://api.ycodify.com/v3/auth/up/sign-in` |
| orgid | `/v3/orgid` | `https://api.ycodify.com/v3/orgid/open/up/account` |
| persistence-crs | `/v3/persistence/c` | `https://api.ycodify.com/v3/persistence/c/a/{bc}/{type}` |
| persistence-q | `/v3/persistence/q` | `https://api.ycodify.com/v3/persistence/q/` |

- O gateway injeta o header **`X-Client: web`** (não é preciso enviá-lo). **`X-Forger-Credential`, ao contrário, NÃO é injetado — o chamador DEVE enviá-lo** (ver seção abaixo).
- **Serviços internos** (atrás do gateway, **sem rota** — não alcançáveis de fora; invocados internamente pela
  plataforma): **`es-n`** (despacho de eventos, acionado pelo crs), **`br-service`** (regras/coordenação `br.route`,
  invocado pelo crs) e **`cache`** (redis — o **forger** o usa: publicar o `.model.json` grava no cache, e criar
  dataschemas para as entidades/projeções é via forger). Não há endpoint público para eles.
- **Persistência — dois contextos** (independentes da base; escolha ortogonal ao ambiente): **prod**
  `/v3/persistence/{c,q}`; **teste** `/v3/persistence/t/{c,q}`. Os paths downstream (`/a/{bc}/{type}`, `/`) são os
  **mesmos** — só muda o segmento `t`.
- Path fora da allowlist `/v3/{auth,orgid,forger,persistence/c,persistence/q,persistence/t/c,persistence/t/q}` → `404`; `/unsecured/**` → `404`.

## `X-Forger-Credential` (credencial do gateway)

- **Obrigatório em TODA requisição ao gateway** — auth, orgid, forger e persistence (c/q, prod/teste).
  O API Gateway rejeita a requisição sem ele.
- Valor: um **UUID** — a credencial do gateway, **fornecida no onboarding** (não é derivável nem escolhida
  pelo chamador). Trate-a como segredo (não commitar; redação em logs).
- ⚠️ **Necessário ANTES do login**: como o próprio `POST /up/sign-in` (ou `/ua/sign-in`) passa pelo gateway,
  ele já exige `X-Forger-Credential`. **Sem esse header não se obtém o token** — logo ele precede a obtenção
  do `Authorization: Bearer`.
- Envie em cada requisição: `-H "X-Forger-Credential: <uuid>"`.

## `Authorization` (token de acesso)

- As requisições autenticadas levam o cabeçalho `Authorization` com um **token de acesso** no esquema
  **portador** (`Authorization: Bearer <token>`).
- A **emissão** do token (login) é feita pelo serviço **[auth](auth/README.md)** — `POST /up/sign-in`
  (plataforma) ou `POST /ua/sign-in` (externo); renovação em `GET /up/sign-in/renew`. Os demais serviços
  apenas **validam** o token recebido (CP-12).
- O token determina **quais organizações/papéis/tenants** o portador possui; isso governa a autorização
  (ex.: papéis exigidos por um comando — ver `roles` em [model-format](persistence-crs/spec/model-format.md#comandos)).

> Obtenha o token autenticando no **[auth](auth/endpoints/sign-in.md)**; depois use-o em
> `Authorization: Bearer <token>` nos demais serviços.

## Autorização (RBAC por papéis)

Autenticação diz **quem é** o portador; a **autorização** diz **o que ele pode** — por **papéis** (roles).
São **dois eixos**, ambos resolvidos comparando os **papéis do JWT** contra uma lista permitida (interseção
vazia → **`403`**):

| Eixo | Onde a lista é declarada | Controla |
|---|---|---|
| **WRITE** (comando no agregado) | `.model.json` → `command.<X>.roles[]` ([model-format](persistence-crs/spec/model-format.md#comandos)) | executar comando via **persistence-crs** |
| **READ** (projeção) | entity `_conf.accessControl.read[]` ([forger/entity](forger/endpoints/entity.md)) | consultar projeção via **persistence-q** |
| **WRITE na projeção** | entity `_conf.accessControl.write[]` | **só a plataforma** (a persistence-crs grava a projeção como `MASTER`; o app **nunca** escreve o read-model direto) |

- **Papéis: UPPERCASE**, **sem** prefixo `ROLE_` (o runtime compara case-sensitive; adiciona o prefixo se
  precisar). Vale em `command.roles[]`, em `accessControl.{read,write}` e na criação de papel.
- **`accessControl` é opcional**: o Forger aplica o default `{read:[MASTER],write:[MASTER]}` em toda entity; se
  **declarado**, `MASTER` deve constar em `read` e `write` (ver [forger/entity](forger/endpoints/entity.md)).
- Os **papéis** (criar `MASTER` e os demais na organização, vincular usuários) são geridos no **[orgid](orgid/README.md)**
  (`POST /ua/role`) — sem isso, ninguém tem authority para operar.

## `X-Tenant-Id` (identificação do tenant)

- O **`tenant-id`** isola o processamento de comandos, eventos e projeções (ver [conceitos](02-conceitos.md)).
- Ele é **gerado na criação do `dataschema`** (forger), não é escolhido pelo chamador.
- **Como obtê-lo:** liste os modelos publicados do projeto — `GET /org/{org}/project/{project}/model`
  ([forger/model](forger/endpoints/model.md)) retorna, por entrada, `{ tenantId, dataschema, project, status, ... }`.
  Alternativamente, leia o recurso `dataschema` correspondente ([forger/dataschema](forger/endpoints/dataschema.md)).
- Trate o `tenant-id` como **opaco** — não interprete sua estrutura.

## Resumo por requisição

| Serviço · operação | `X-Forger-Credential` | `Authorization` | `X-Tenant-Id` |
|---|---|---|---|
| auth · `up/sign-in` · `ua/sign-in` | **sim** | não (é quem emite) | não |
| forger · recursos/model/process | **sim** | sim | não (usa `{org}`/`{project}`/`{tenantId}` no caminho) |
| persistence-crs · comando/leitura/projeção | **sim** | sim | **sim** |
| persistence-q · consulta | **sim** | sim | **sim** |
| br-service · `/br` | (interno) | (chamado pelo persistence-crs) | propagado pelo chamador |
| es-n · acionar processamento | (interno) | operacional/interno | tenant no caminho |
