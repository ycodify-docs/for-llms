# persistence-q · exemplos anotados

> Dados genéricos. Cabeçalhos `Authorization` + `X-Tenant-Id` em todas. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.

> ⚠️ **Rótulo = nome da projeção**: os rótulos abaixo (`pedidosCriados` etc.) são ilustrativos — na prática
> o rótulo DEVE ser o **nome da projeção/entidade** provisionada (ex.: `pedido`); rótulo arbitrário →
> `510` (projeção de nome `<rótulo>` inexistente). Ver [endpoints/consulta.md](endpoints/consulta.md).

> ⚠️ **Dois ids na projeção**: cada linha tem `id` (**Long**, PK da linha) e `aggregateid` (**UUID**, o
> id do agregado). Nas respostas abaixo o `<uuid>` é o **`aggregateid`**. Para localizar **um** agregado
> filtra-se por `aggregateid`; para operar o agregado depois (comando/`/history` na persistence-crs)
> usa-se o **`aggregateid`** (UUID), nunca a PK `id` (Long). Ver
> [model-format §Identificação na projeção](../persistence-crs/spec/model-format.md).

## Critério único (modo object)

```
POST /
{ "pedidosCriados": { "status": "criada" } }

→ 200
[ { "pedidosCriados": [ { "id": 42, "aggregateid": "<uuid>", "status": "criada", "total": 25000 } ] } ]
```

## Múltiplos critérios (modo array)

```
POST /
[
  { "pedidoPorId": { "aggregateid": "<uuid>" } },
  { "clientesAtivos": { "ativo": true } }
]

→ 200
[
  { "pedidoPorId":   [ { "id": 42, "aggregateid": "<uuid>", "status": "confirmada" } ] },
  { "clientesAtivos": [ { "id": 7, "aggregateid": "<uuid>" }, { "id": 9, "aggregateid": "<uuid>" } ] }
]
```

## Sem resultados

```
POST /
{ "pedidoPorId": { "aggregateid": "<uuid-inexistente>" } }

→ 204   // nenhum resultado (não é erro)
```

## Predicado desconhecido

```
POST /
{ "x": { "campoQueNaoExiste": 1 } }

→ 510   // predicado fora do vocabulário do tenant
```

> O conjunto de rótulos e predicados válidos vem do **model**/projeções do tenant (ver
> [examples/](../examples/README.md)). A leitura reflete o estado materializado pelo fluxo
> es-n → projeção, que é **assíncrono** após cada comando.
