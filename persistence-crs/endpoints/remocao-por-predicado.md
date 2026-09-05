# persistence-crs · endpoints · remoção por predicado

> Remove **N linhas** de uma entidade pelo critério que você informar — não pela chave primária. Rota
> própria, separada do [`/e`](entidade.md), porque o risco é outro: aqui um filtro errado alcança muitas
> linhas de uma vez. Por isso a rota **conta antes**, exige um teto e, por padrão, **não apaga na primeira
> chamada**.
> Pré-requisitos: [conceitos](../../02-conceitos.md), [autenticação](../../06-autenticacao.md).
> Guia: [../README.md](../README.md). Para apagar **uma** linha por `id`, use [entidade](entidade.md).

> **URL externa.** `POST /e/by` é o path downstream; via gateway: **produção**
> `POST /v3/persistence/c/e/by` · **teste** `POST /v3/persistence/t/c/e/by`.

## Quando usar

| Você quer… | Use |
|---|---|
| apagar **uma** linha, que você identifica por `id` | `POST /e` com `action: DELETE` ([entidade](entidade.md)) |
| apagar **as linhas que casam um critério** | `POST /e/by` (este doc) |

⚠️ Vale aqui a mesma advertência de [entidade](entidade.md): esta rota **não é comando**. Não passa pelo
RBAC de comando nem pelo processor `br`, e a projeção de um agregado escrita por aqui fica fora do log de
eventos.

## Requisição

`POST /e/by`

Cabeçalhos: `Authorization` + `X-Tenant-Id`; `Content-Type: application/json`.

> ⚠️ Bloco **ilustrativo**: `<...>` são placeholders, não JSON literal pronto para envio.

```json
{
  "entity":      "<nome da entity>",
  "where":       { "<atributo>": "<valor>" },
  "_connective": "AND",
  "_maxRows":    50,
  "_dryRun":     true
}
```

| Campo | Obrigatório | O que é |
|---|---|---|
| `entity` | **sim** | a entidade cujas linhas serão removidas |
| `where` | **sim**, e **não pode ser vazio** | o critério. Mesmos operadores da [consulta](../../persistence-q/query-controls.md) |
| `_connective` | **sim** | `AND` ou `OR` — como os critérios se combinam. **Explícito de propósito**: no `/e` o `AND` é implícito e inverte o resultado de quem esperava `OR` |
| `_maxRows` | **sim**, `> 0` | quantas linhas você **aceita** apagar. Se o filtro casar mais que isso, a resposta é `409` e **nada é apagado** |
| `_dryRun` | não — **default `true`** | `true` conta e mostra, sem apagar. Para apagar, mande `false` |

## Respostas

| Situação | Código | Corpo |
|---|---|---|
| ensaio (`_dryRun: true`) | `200` | `{"dryRun":true,"entity":…,"matched":N,"maxRows":M,"deleted":0}` |
| removeu | `200` | `{"dryRun":false,"entity":…,"matched":N,"deleted":N}` |
| o filtro casa mais que o teto | `409` | `{"error":…,"entity":…,"matched":N,"maxRows":M}` — **nada foi apagado** |
| corpo inválido | `400` | `{"error":…}` com o que falta |
| tenant não autorizado | `403` | — |
| falha de aplicação | `510` | ex.: violação de restrição |

## Comportamento

- **A contagem vem do mesmo comando que apagaria.** O serviço monta o `DELETE`, converte-o em
  `SELECT count(*)` e é esse número que aparece em `matched` — ele não pode divergir do que seria
  removido.
- **`where` vazio é recusado.** Não existe "apague tudo" por omissão nesta rota.
- **Critério que não monta filtro nenhum também é recusado** (`400`) — por exemplo, um atributo que não
  existe no modelo da entity. Sem essa recusa o comando sairia sem `WHERE`.
- A remoção é aplicada **na requisição**, direto na tabela do tenant: sem comando, sem evento e sem
  validação de transição de estado.
- Só entidade **relacional**. Entidade não relacional é recusada com `400`.

## Autorização — leia antes de expor esta rota

A mesma do [`/e`](entidade.md): pertencer ao tenant **e** ter papel em `_conf.accessControl.write` da
entity. **Não há escopo por linha** — a checagem olha o papel de quem chama e nunca a linha alvo, então
quem pode apagar uma linha pode apagar as de todas as pessoas. Regra de dono ("só o titular mexe no seu
registro") vive em processor `br`, e esta rota não o executa.

Trate `POST /e/by` como operação administrativa: exponha-a a quem você exporia o banco.

## Erros

`400` (corpo inválido; `where` vazio; `_connective` ausente ou diferente de `AND`/`OR`; `_maxRows`
ausente ou `<= 0`; critério que não monta filtro; entidade não relacional), `403` (tenant),
`409` (teto excedido — nada apagado), `510` (falha de aplicação).
Catálogo: [../erros.md](../erros.md).
