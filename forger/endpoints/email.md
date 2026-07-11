# forger · endpoints · email

> **email** = configuração de envio de e-mail vinculada a um **project**. Guia: [../README.md](../README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Criar

`POST /org/{org}/project/{project}/email`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | sim | Nome/identificação da configuração. |
| `token` | string | sim | Credencial do provedor de envio. |
| `status` | string | sim | Situação. |

Resposta `201`: `{ "id": <número>, "message": "..." }`. Erros: `400`, `403`, `500`.

## Ler / Listar
- `GET /org/{org}/project/{project}/email/{id}` → `200`/`404`.
- `GET /org/{org}/project/{project}/email` → `200` array.

## Atualizar
`PUT /org/{org}/project/{project}/email/{id}` — corpo: `logversion` + (`name`, `token`, `status`).
`200`; conflito → `409`.

## Remover
`DELETE /org/{org}/project/{project}/email/{id}` → `200`/`404`.
