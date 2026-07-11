# Autenticação e identificação de tenant

> Como preencher os dois itens que **toda** requisição de comando/consulta exige: o cabeçalho
> `Authorization` e o cabeçalho de tenant `X-Tenant-Id`. Pré-requisitos: [conceitos](02-conceitos.md).

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

## `X-Tenant-Id` (identificação do tenant)

- O **`tenant-id`** isola o processamento de comandos, eventos e projeções (ver [conceitos](02-conceitos.md)).
- Ele é **gerado na criação do `dataschema`** (forger), não é escolhido pelo chamador.
- **Como obtê-lo:** liste os modelos publicados do projeto — `GET /org/{org}/project/{project}/model`
  ([forger/model](forger/endpoints/model.md)) retorna, por entrada, `{ tenantId, dataschema, project, status, ... }`.
  Alternativamente, leia o recurso `dataschema` correspondente ([forger/dataschema](forger/endpoints/dataschema.md)).
- Trate o `tenant-id` como **opaco** — não interprete sua estrutura.

## Resumo por requisição

| Serviço · operação | `Authorization` | `X-Tenant-Id` |
|---|---|---|
| forger · recursos/model/process | sim | não (usa `{org}`/`{project}`/`{tenantId}` no caminho) |
| persistence-crs · comando/leitura/projeção | sim | **sim** |
| persistence-q · consulta | sim | **sim** |
| br-service · `/br` | (chamado pelo persistence-crs) | propagado pelo chamador |
| es-n · acionar processamento | operacional/interno | tenant no caminho |
