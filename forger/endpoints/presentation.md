# forger · endpoints · presentation

> **presentation** = o **manifesto de apresentação** de um tenant: como a tela mostra os agregados —
> rótulos, formatos, colunas, filtros, opções e cor de cada estado. **Opcional**: sem ele, a tela deriva
> tudo do nome e do tipo. Guia: [../README.md](../README.md). Forma: [presentation.schema.json](../spec/presentation.schema.json).
> Consumo pela tela: [bff](../../bff/README.md).

## Contents
- Publicar · Ler · Remover
- Forma do arquivo
- Referências conferidas na publicação
- Ciclo de vida

Um manifesto por tenant, publicado **depois** do `.model.json` daquele tenant: é contra o modelo
publicado que as referências são conferidas. O motor de execução **não** lê este arquivo.

## Publicar

`POST /org/{org}/project/{project}/tenant/{tenantId}/presentation`

Envio **multipart**, com o documento no campo `file` e extensão `.json`. Autorização: a mesma do
[model](model.md) — papel de administrador ou de engenheiro em `{org}`.

Efeito, na ordem: confere que `(org, project, tenant)` referenciam um único dataschema → remove as chaves
`_`-prefixadas (metadado) → valida a **forma** → exige o **modelo publicado** do tenant → confere as
**referências** → publica. **Semântica de sobrescrita.** Um manifesto recusado **não** substitui o que
estava publicado.

| Resposta | Quando |
|---|---|
| `201` `{ "stored": true }` | publicado |
| `400` | arquivo vazio ou sem `.json`; forma inválida; modelo do tenant não publicado; referência inexistente. Traz **todas** as violações de uma vez, cada uma nomeando o que está errado |
| `403` · `404` | consistência de `(org, project, tenant)`, como no model |

## Ler

`GET /org/{org}/project/{project}/tenant/{tenantId}/presentation`

**Lê quem tem o tenant no token** — sem exigir papel de administrador nem de engenheiro, que é a exceção
à regra geral do serviço — **ou** quem tem papel de administrador ou de engenheiro em `{org}`: é assim que
quem publica confere o que publicou, porque o token de plataforma não carrega os tenants.

| Resposta | Quando |
|---|---|
| `200` | o manifesto, como publicado (sem as chaves `_`) |
| `204` sem corpo | o tenant não tem manifesto — é o caso normal, e a tela deriva do nome e do tipo |
| `403` | o tenant não está no token, e o usuário não tem papel de administrador nem de engenheiro em `{org}` |
| `404` | tenant inexistente ou de outro project |

## Remover

`DELETE /org/{org}/project/{project}/tenant/{tenantId}/presentation` — autorização do publicar.
`200` `{ "deleted": true }` — o corpo diz o **estado**: não resta manifesto, tivesse ou não havido um.
Repetir o `DELETE` é seguro e responde o mesmo.

## Forma do arquivo

```json
{
  "formato": 1,
  "aggregate": {
    "loja.pedido": {
      "singular": "Pedido", "plural": "Pedidos", "titleKey": "numero",
      "labels": { "numero": "Número", "endereco.cep": "CEP" },
      "stateLabels": { "criado": "Criado", "pago": "Pago" },
      "fmt": { "total": "money:2", "criadoem": "datetime" },
      "cols": ["numero", "total"], "filters": ["numero"],
      "options": { "canal": ["loja", "site"],
                   "cliente": { "aggregate": "loja.cliente", "valueKey": "id", "labelKey": "nome" } },
      "stateHue": { "criado": 210, "pago": 120 }
    }
  }
}
```

> Ilustrativo: nomes de agregado, atributo e estado são exemplos, não JSON literal a copiar.

`formato` é **obrigatório** e vale `1`. Todo o resto é **opcional**. A chave de `aggregate` é a **mesma**
do `.model.json` (`<boundedContext>.<type>`). Chave desconhecida é **recusada** — typo não passa em silêncio.

| Chave | O que é |
|---|---|
| `singular` · `plural` | nome do agregado na tela; texto não vazio |
| `titleKey` | atributo cujo valor identifica a linha |
| `labels` | rótulo por atributo, value object ou `grupo.campo` |
| `stateLabels` | rótulo por estado |
| `fmt` | formato por atributo: `date` · `datetime` · `money` · `money:<casas>` · `phone` · `bool` · `mono`. `money:2` é inteiro com 2 casas implícitas (`1050` = `10,50`) |
| `cols` · `filters` | atributos das colunas padrão, na ordem, e atributos filtráveis; lista não vazia, sem repetição |
| `options` | por atributo, lista fixa de valores **ou** `{ aggregate, valueKey, labelKey }` apontando outro agregado |
| `stateHue` | matiz da cor de cada estado, inteiro de `0` a `359` |

O nome de exibição de **comando** e de **evento** não entra aqui: é o `alias`, declarado no próprio modelo
([model](model.md#nome-de-exibicao)).

## Referências conferidas na publicação

Toda referência tem de existir no **modelo publicado do mesmo tenant**:

- **agregado** — chave de `aggregate` do modelo;
- **atributo** — declarado em `data.attribute` de algum comando do agregado, ou `whenAttribute` de algum
  evento dele, ou chave da plataforma — `id`, `aggregateid`, `status`, `version` e os metadados de auditoria
  (`loguser`, `logrole`, `logversion`, `logdate`), a lista do [model-format](../../persistence-crs/spec/model-format.md).
  Para `options` por referência, o `valueKey` costuma ser `aggregateid`, que é o que um campo de referência guarda;
  o `id` é a chave da linha na projeção e não serve de referência;
- **`grupo.campo`** — só em value object que é **grupo de campos**; value object descrito como um
  atributo não tem campos para apontar;
- **value object pelo nome** — só em `labels`;
- **estado** (`stateLabels`, `stateHue`) — chave de evento do agregado;
- **`options` por referência** — outro agregado do **mesmo** modelo, com `valueKey` e `labelKey` entre os
  atributos **dele**.

## Ciclo de vida

- O manifesto **não** é removido junto com o modelo — nem pelo `DELETE` do model, nem quando o dataschema
  entra em `MODELING`. Referência que ficar órfã (atributo renomeado no modelo) a tela ignora, e a
  próxima publicação do manifesto recusa.
- Chaves `_`-prefixadas são metadado, removidas na publicação, como no `.model.json`.

> **Em produção desde 2026-09-26** (ver [CHANGELOG 1.61](../../CHANGELOG.md)). Até o próximo deploy, três
> diferenças em produção: o `DELETE` ainda responde `{ "deleted": false }` quando não havia manifesto —
> inclusive numa repetição da mesma chamada, com o manifesto já apagado pela primeira —; a publicação
> ainda recusa `aggregateid`, `status` e `version` como atributo; e o `GET` ainda responde `403` a quem só
> tem papel de administrador ou de engenheiro, sem o tenant no token.
