# forger · endpoints · project

> **project** = um **bounded context** (contexto delimitado, DDD). Por convenção padrão, **1 project =
> 1 bounded context** ([topologia padrão](../../02-conceitos.md#convenção-padrão-de-topologia-project--bounded-context--esquema)).
> Criar um project gera os identificadores de tenant associados ao contexto. Guia: [../README.md](../README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Criar

`POST /org/{org}/project`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | sim | Nome do contexto. Só letras minúsculas (`^[a-z]+$`), **máx. 16 caracteres**. |
| `alias` | string | sim | Apelido legível. |
| `envtype` | string | sim | Tipo de ambiente. |
| `email` | string | não | Contato. |
| `description` | string | não | Descrição. |
| `status` | string | não | Situação inicial. |

Resposta `201`: `{ "id": <número>, "tenantSecret": "<uuid>", "tenantPid": "<uuid>", "message": "..." }`.
Os identificadores de tenant retornados pertencem ao contexto. Erros: `400`, `403`, `500`.

## Ler / Listar

- `GET /org/{org}/project/{project}` → `200`/`204`.
- `GET /org/{org}/project` → `200` array; `204` vazio.

## Atualizar

`PUT /org/{org}/project/{project}` — corpo: `logversion` + editáveis (`alias`, `envtype`, `email`,
`description`, `status`). `200`; conflito → `409`.

## Remover

`DELETE /org/{org}/project/{project}` → `200`/`204`. **Bloqueado** se houver **dataschema** no contexto.

## Coordenação
Necessário (junto com **database**) para criar **dataschema** ([dataschema.md](dataschema.md)).
