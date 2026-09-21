# persistence-crs · endpoints · leitura de agregado

> Leitura do **estado atual** e do **histórico de eventos** de um agregado, direto pelo lado de escrita.
> Para consultas por critério sobre projeções, use [persistence-q](../../persistence-q/README.md).
> Guia: [../README.md](../README.md).

Cabeçalhos: `Authorization` + cabeçalho de tenant `X-Tenant-Id`.

## Estado atual

`GET /a/{boundedContext}/{aggregateType}/{id}`

- `200` — estado atual do agregado (JSON), reconstituído a partir dos seus eventos.
- `204` — agregado não encontrado.

**Datas no estado:** `Timestamp` vem como `"2026-09-15T09:00:00"` — **UTC, sem fuso** — e `Date` como
`"1990-04-02"`. Inclusive o carimbo automático de cada evento (o `whenAttribute`, como `criadaem`):

```json
{ "id": "<uuid>", "status": "cadastrada",
  "datahorainicio": "2026-09-15T09:00:00",
  "cadastradaem":   "2026-09-13T20:19:09" }
```

**É a mesma string que a projeção devolve** para o mesmo campo — pode comparar sem converter. Para mostrar
no horário de Brasília, a conversão é do front (exemplo em
[persistence-q § Resposta](../../persistence-q/endpoints/consulta.md#resposta)).

> Antes de `yc-interpreter:amd64-260913c` — em teste desde essa imagem, **produção ainda não** —, este
> endpoint devolvia a data **como o cliente a
> mandou** — com fuso, fração ou espaço — e o carimbo automático com espaço (`2026-09-13 20:19:09`).

## Histórico de eventos

`GET /a/{boundedContext}/{aggregateType}/{id}/history`

- `200` — lista (array) dos eventos do agregado, em ordem.
- `204` — sem eventos.

> **`{id}` = UUID do agregado** (o `aggregateid` da projeção; `projecao.aggregateid == aggregate.id`).
> **Não** use a PK `id` (Long) da projeção — enviar o Long → `510` "Invalid UUID string".

## Quem pode ler

Por padrão, **qualquer usuário autenticado do tenant** lê qualquer agregado dele: os dois endpoints
conferem o tenant, e nada mais.

Quando o modelo do agregado declara **`roles`**
([model-format](../spec/model-format.md#quem-pode-ler-o-agregado-roles)), os dois passam a recortar:

| Situação | Resposta |
|---|---|
| papel do solicitante fora de `roles.read` | **`204`** — a mesma resposta de um `{id}` que não existe |
| recorte por proprietário, e o agregado é de outra pessoa | **`204`** |
| passa no recorte | `200` normal — e o `/history` vem **inteiro** |
| passa, mas o papel não está em `roles.author` | `200`, com os eventos **sem o bloco de autoria** |

> **`204` aqui não distingue "não existe" de "não é seu", e é de propósito:** um `403` entregaria a
> existência do agregado a quem não pode vê-lo. Ao depurar um `204` inesperado, verifique o recorte do
> modelo antes de procurar o dado.

**`loguser` aparece nas duas respostas** — no estado e em cada evento — com o `username` de quem executou
aquele comando. É carimbado pela plataforma: não se declara no modelo nem se envia no comando. Em comando
disparado por coordenação, é o autor do **primeiro** comando da cadeia.

## Quando usar

- **Estado atual:** inspeção pontual de um agregado conhecido pelo seu `id` (UUID do agregado).
- **Histórico:** auditoria/depuração da sequência de eventos (event sourcing).
- Para **buscar** por atributos ou listar muitos registros, use a **consulta** de projeções em
  [persistence-q](../../persistence-q/README.md).

## Erros
`403` (tenant não autorizado), `510` (falha de processamento). Catálogo: [../erros.md](../erros.md).
