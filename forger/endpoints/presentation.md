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
      "singular": "Pedido", "plural": "Pedidos", "genero": "masculino", "titleKey": "numero",
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
| `genero` | gênero gramatical desse nome, para a tela concordar ("Nova modalidade"): `feminino` ou `masculino`; ausente = masculino |
| `titleKey` | atributo (ou `grupo.campo`) cujo valor identifica a linha |
| `labels` | rótulo por atributo, `grupo.campo` ou value object |
| `stateLabels` | rótulo por estado |
| `fmt` | formato por atributo ou `grupo.campo`: `date` · `datetime` · `money` · `money:<casas>` · `phone` · `bool` · `mono`. `money:2` é inteiro com 2 casas implícitas (`1050` = `10,50`) |
| `cols` · `filters` | colunas padrão, na ordem, e campos filtráveis — atributo ou `grupo.campo`; lista não vazia, sem repetição |
| `options` | por atributo ou `grupo.campo`, lista fixa de valores **ou** `{ aggregate, valueKey, labelKey }` apontando um agregado |
| `stateHue` | matiz da cor de cada estado, inteiro de `0` a `359` |

O nome de exibição de **comando** e de **evento** não entra aqui: é o `alias`, declarado no próprio modelo
([model](model.md#nome-de-exibicao)).

## Referências conferidas na publicação

Toda referência tem de existir no **modelo publicado do mesmo tenant**. Cinco tipos:

- **agregado** — chave de `aggregate` do modelo;
- **atributo** — declarado em `data.attribute` de algum comando do agregado, ou `whenAttribute` de algum
  evento dele, ou chave da plataforma — `id`, `aggregateid`, `status`, `version` e os metadados de auditoria
  (`loguser`, `logrole`, `logversion`, `logdate`), a lista do [model-format](../../persistence-crs/spec/model-format.md);
- **`grupo.campo`** — campo de um value object (`single` ou `multiple`) que é **grupo de campos**; value
  object descrito como um atributo (com `type` no próprio nó) não tem campos para apontar;
- **value object pelo nome** — o nome do grupo sozinho;
- **estado** — chave de evento do agregado.

O que cada chave aceita — é exatamente o que a publicação confere; o resto é `400`:

| Chave | Aceita |
|---|---|
| `titleKey` | atributo ou `grupo.campo` |
| `labels` (chave) | atributo, `grupo.campo` ou value object pelo nome — **só aqui** o nome do grupo sozinho vale |
| `fmt` (chave) | atributo ou `grupo.campo` |
| `cols` · `filters` (item) | atributo ou `grupo.campo` |
| `options` (chave) | atributo ou `grupo.campo` |
| `options` por referência: `aggregate` | agregado do mesmo modelo, inclusive o próprio |
| `options` por referência: `valueKey` · `labelKey` | atributo ou `grupo.campo` **do agregado apontado** |
| `stateLabels` · `stateHue` (chave) | estado |

Em `options` por referência, o `valueKey` costuma ser `aggregateid`, que é o que um campo de referência
guarda; o `id` é a chave da linha na projeção e não serve de referência.

## Ciclo de vida

- O manifesto **não** é removido junto com o modelo — nem pelo `DELETE` do model, nem quando o dataschema
  entra em `MODELING`. Referência que ficar órfã (atributo renomeado no modelo) a tela ignora, e a
  próxima publicação do manifesto recusa.
- Chaves `_`-prefixadas são metadado, removidas na publicação, como no `.model.json`.

> **Em produção desde 2026-09-26** (ver [CHANGELOG 1.68](../../CHANGELOG.md)), incluindo o `DELETE` que responde o
> estado, as chaves da plataforma do canon e o `GET` para papel de administrador ou de engenheiro. **A chave
> `genero` está em `develop` do forger desde 2026-09-27 e ainda não em produção**: até o próximo deploy, a
> publicação a recusa como chave desconhecida (`400`).
