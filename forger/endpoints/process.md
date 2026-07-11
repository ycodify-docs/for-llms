# forger · endpoints · process

> **process** = modelo de **processo/orquestração** (e decisão) de um bounded context, publicado para
> execução. Opcional: só relevante quando o sistema usa processos. Guia: [../README.md](../README.md).

Exigem `Authorization` e papel de administrador/engenheiro em `{org}`.

## Publicar

`POST /org/{org}/project/{project}/process`

Envio: o diagrama de processo como upload. Parâmetros de consulta opcionais controlam **validação** e o
tratamento de elementos órfãos. Efeito:
1. transforma o diagrama em representação executável;
2. faz **análise de qualidade** — se houver problemas **críticos**, a publicação é **bloqueada** com
   `422` e o relatório de análise é retornado;
3. **provisiona a topologia de mensageria** declarada no diagrama (ver abaixo);
4. publica no cache distribuído (**sobrescreve** o anterior).

Resposta `201`: `{ "key": "<referência>", "analysis": { ... }, "brokers": [ ... ] }`.
Erros: `422` (problemas críticos — não publica), `409` (conflito de versão de topologia), `400`, `500`.

## Provisionamento de filas (topologia de mensageria)

**É aqui — no deploy de `process`, não no de `model` — que as filas são criadas.** O diagrama declara,
em seus elementos de extensão, uma ou mais **unidades de topologia**. Cada unidade tem:

| Campo | Significado |
|---|---|
| `exchange` | ponto de troca para onde as mensagens são publicadas |
| `queue` | fila principal |
| `dlqueue` | fila de descarte (dead-letter) associada |
| `routekey` | chave(s) de roteamento que ligam a fila ao ponto de troca |

Para cada unidade declarada, o forger:
1. registra/atualiza os metadados da topologia (catálogo de filas do projeto);
2. **declara** a topologia no **broker de mensagens**;
3. responde no array `brokers` um resumo por fila: `{ id, exchange, queue, dlqueue, routekey, status, operation }`,
   onde `operation` é `created` ou `updated`.

Semântica:
- **UPSERT por fila** — fila inédita → criada; fila já existente → atualizada (com **versão otimista**:
  conflito → `409`).
- **Órfãos** — se o parâmetro de inativação estiver ligado, filas antes declaradas e **ausentes** no
  novo diagrama são marcadas como inativas.
- **Compensação** — se a declaração no broker falhar, o registro recém-inserido é revertido e o deploy
  é abortado (não fica topologia parcial).

## Ler

`GET /org/{org}/project/{project}/process` → `200` com o processo publicado; `404` se ausente.

## Remover

`DELETE /org/{org}/project/{project}/process` → `200` `{ "deleted": true }`.

## Ciclo de vida
Auth → transformação → **portão de qualidade** (crítico bloqueia, `422`) → provisionamento de
mensageria → publicação no cache → resposta.

> Endpoints de **transformação sem autenticação** (conversão de diagrama sem publicar) existem para uso
> interno e **não** fazem parte desta documentação pública.
