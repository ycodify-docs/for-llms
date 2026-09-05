# forger · endpoints · dbconn

> **dbconn** = metadados de conexão a um **servidor de banco relacional** do cliente. Primeiro recurso
> a existir no [fluxo de deploy](../../03-fluxo-de-deploy.md). Guia do serviço: [../README.md](../README.md).

Todos os endpoints exigem `Authorization` e papel de administrador/engenheiro na organização `{org}`.

## Criar

`POST /org/{org}/dbconn`

Corpo (JSON):

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dbsqlserveraddress` | string | sim | Endereço do servidor de banco relacional. |
| `dbsqlname` | string | sim | Nome lógico da conexão. |
| `dbsqlserverport` | inteiro (>0) | sim | Porta do servidor. |
| `dbsqlusername` | string | sim | Usuário de conexão. |
| `dbsqluserpassword` | string | não | Senha de conexão. |

Resposta `201`: `{ "id": <número>, "message": "..." }`.
Erros: `400` (campo ausente/ inválido), `403`, `500`.

> ⚠️ **Alterar uma conexão não passa a valer de imediato.** Os serviços de comando e de consulta
> guardam a conexão resolvida do tenant em memória por um intervalo; até ele vencer, as requisições
> continuam usando os **dados antigos** — inclusive usuário e senha anteriores. Consequência prática:
> depois de trocar endereço, porta ou credencial aqui, **o efeito pode demorar** e uma falha de
> autenticação logo após a troca **não** significa que o novo valor está errado. Planeje a troca com
> essa defasagem em mente.

## Ler

`GET /org/{org}/dbconn/{id}` → `200` com o registro; `204` se não existir.

## Listar

`GET /org/{org}/dbconn` → `200` com array de registros; `204` se vazio.

## Atualizar

`PUT /org/{org}/dbconn/{id}`

Corpo: `logversion` (inteiro, **versão otimista atual**) + os campos editáveis
(`dbsqlserveraddress`, `dbsqlserverport`, `dbsqlusername`, `dbsqluserpassword`).
Resposta `200`. Conflito de versão → `409`.

## Remover

`DELETE /org/{org}/dbconn/{id}` → `200`/`204`.
**Bloqueado** (`409`/erro) se houver **databases** vinculados a esta conexão.

## Ciclo de vida

Operação só de metadados (não cria banco físico aqui — isso ocorre ao criar um **database**). Segue o
[ciclo de vida geral](../README.md#ciclo-de-vida-geral-de-uma-requisição): auth → validação → efeito → resposta.

## Coordenação
Habilita a criação de **database** ([database.md](database.md)), passo seguinte do deploy.
