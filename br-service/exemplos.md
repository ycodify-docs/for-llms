# br-service · exemplos anotados

> O br-service é chamado pelo persistence-crs. Os exemplos mostram o **contrato** e a **forma de um
> processador**. Dados genéricos. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.

## Chamada (como o persistence-crs invoca)

```
POST /br
{ "route": "vendas/pedido/validar", "data": { "total": 250.0, "cliente": "C-1" } }

→ 200
{ "total": 250.0, "cliente": "C-1", "descontoAplicado": 0.0, "valido": true }
```

A resposta é o objeto **direto** do processador; o persistence-crs a usa para enriquecer/validar o comando.

As três categorias diferem pela **rota** (segmento de path) e pelo que retornam em `processedData`
(ver [README — três categorias](README.md#três-categorias-de-processador)). O retorno real costuma vir
embrulhado em `{ message, processedData, receivedData, processedBy }`; o que importa para o fluxo é
`processedData`.

### 1) Regra de negócio — rota `…/<agregado>/<comando>`

Valida/enriquece os dados do comando; `processedData` volta para o comando antes do evento.
`authToken` (JWT do usuário) e `tenantIds` chegam como **argumentos separados** quando presentes.

```js
// Hook SÍNCRONO — assinatura completa
async function(data, authToken, tenantIds) {
  // data = payload bruto do comando (atributos de command.<cmd>.data.attribute[])
  // authToken = JWT do usuário (string) ou undefined se nulo/ausente
  // tenantIds = [tenantId.forReadModel] do modelo — NÃO entregue se authToken for nulo
  if (!data.cliente) throw new Error('cliente obrigatório');
  return { processedData: { ...data, cliente: data.cliente.trim() } };
}
```

### 2) Coordenação (saga) — rota `…/<agregado>/coordination/<alvo>_from_<origem>`

Recebe dados do evento + estado do agregado de origem; monta um **`targetCommand`** para o contexto destino.
`authToken` chega na raiz do corpo → entregue como **segundo argumento** se não nulo.

```js
// Hook ASSÍNCRONO — data tem forma específica (ver README § Forma do data)
async function(data, authToken) {
  // data.<entityName>   = dados do evento de origem
  // data.aggregateState = estado completo do agregado de origem (carregado pelo crs)
  // data.targetTenantId / data.sourceTenantId — top-level em data (NÃO em _meta)
  // authToken = JWT de origem propagado (pode ser undefined — trate como oportunista)
  return { processedData: { targetCommand: {
    boundedContext: 'financeiro',
    aggregateType:  'conta',
    commandName:    'criar',
    data: { razaosocial: data.aggregateState.razaosocial, cnpj: data.aggregateState.cnpj }
  } } };
}
```

O persistence-crs submete esse `targetCommand` como **novo comando** (CP-7).

### 3) Projeção cross-contexto — rota `…/<agregado>/projection/<alvo>_from_<origem>`

Mapeia o evento de origem na **linha da projeção** de outro contexto; pode ler a projeção destino
(somente leitura) para um UPDATE coerente.
`authToken` — quando propagado — chega em **`data._meta.authToken`** (não como argumento).

```js
// Hook ASSÍNCRONO — data tem forma específica (ver README § Forma do data)
async function(data) {
  // data.<entityName>        = dados do evento de origem
  // data._meta.targetTenantId / data._meta.sourceTenantId — dentro de _meta (≠ coordenação)
  // data._meta.authToken     = JWT de origem se propagado; pode estar ausente
  const authToken = data._meta?.authToken;  // oportunista
  return { processedData: { 'demandadecisao': {
    aggregateid:  data.demandadecisao.demandaid,
    decisao:      data.demandadecisao.decisao,
    registradaem: data.demandadecisao.registradaem
  } } };
}
```

## Rota desconhecida

```
POST /br
{ "route": "rota/inexistente", "data": {} }

→ 400 { "status": "error", "mensagem": "Invalid route ... Available routes: ...", "tipo": "Error" }
```

> A rota referenciada deve coincidir com a declarada no **model** do comando/evento (ver
> [examples/](../examples/README.md) e [forger/model](../forger/endpoints/model.md)).
