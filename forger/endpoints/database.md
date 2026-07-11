# forger · endpoints · database

> **database** = um **banco de leitura** vinculado a um `dbconn`. Criar um database **materializa o
> banco físico** no servidor relacional. Guia: [../README.md](../README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Criar

`POST /org/{org}/dbconn/{dbconnId}/database`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dbsqlname` | string | sim | Nome do banco. Só letras minúsculas (`^[a-z]+$`); sem limite de comprimento definido. |
| `dbsqlversion` | string | não | Versão/identificação opcional. |

Efeito: cria os metadados **e** o **banco físico** + papel de acesso no servidor relacional; se a
criação física falhar, os metadados são revertidos (rollback).
Resposta `201`: `{ "id": <número>, "dbsqlname": "...", "dbsqlusername": "...", "message": "..." }`.
Erros: `400` (nome fora do formato/ausente), `403`, `500`.

## Ler

- `GET /org/{org}/dbconn/{dbconnId}/database/{id}` → `200`/`204`.
- `GET /org/{org}/dbconn/{dbconnId}/database/name/{name}` → `200`/`204` (busca por nome).

## Listar

`GET /org/{org}/dbconn/{dbconnId}/database` → `200` array; `204` vazio.

## Atualizar

`PUT /org/{org}/dbconn/{dbconnId}/database/{id}` — corpo: `logversion` + campos editáveis
(`dbsqluserpassword`, `dbsqlversion`). `200`; conflito → `409`.

## Remover

`DELETE /org/{org}/dbconn/{dbconnId}/database/{id}` → `200`/`204`.
**Bloqueado** se houver **dataschema** vinculado.

## Ciclo de vida
Metadados + efeito físico transacional (criação do banco). Ver [ciclo de vida geral](../README.md#ciclo-de-vida-geral-de-uma-requisição).

## Coordenação
Necessário (junto com **project**) para criar **dataschema** ([dataschema.md](dataschema.md)).
