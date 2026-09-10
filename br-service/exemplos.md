# br-service · exemplos anotados

> **Um exemplo por contexto de invocação**, e cada um amarrado a um modelo que já existe em
> [examples/](../examples/README.md) — para que você possa abrir o modelo, publicar o processador e
> repetir. Regra de cada contexto: [contextos.md](contextos.md). Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `…` são placeholders, não JSON literal pronto para envio.

## Contents
- 1 · Regra de negócio
- 2 · Coordenação (saga)
- 3 · Projeção cross-contexto
- Rota desconhecida
- Corpo inválido

---

## 1 · Regra de negócio — `acme.financeiro`

**Onde o gancho está declarado** — [`examples/acme.financeiro.model.json`](../examples/README.md),
agregado `financeiro.conta`, comando `criar`:

```json
"criar": {
  "data": { "attribute": { "razaosocial": {…}, "cnpj": {…}, "email": {…}, "status": {…} } },
  "roles": ["MASTER", "GERENTEFINANCEIRO", "VENDEDOR"],
  "br": { "route": "acme/financeiro/financeiro/conta/criar" }
}
```

**O que chega no processador:**

```json
{
  "route": "acme/financeiro/financeiro/conta/criar",
  "data": {
    "razaosocial": "  Acme Indústria Ltda  ",
    "cnpj": "00.000.000/0001-00",
    "email": "contato@exemplo.invalid",
    "status": "criada"
  },
  "authToken": "Bearer <token do usuário>",
  "tenantIds": ["00000000-0000-4000-8000-000000000002"]
}
```

Três argumentos: `(data, authToken, tenantIds)`. O `tenantIds` traz **um** elemento, o tenant do modelo
de leitura declarado no agregado.

**O que o processador devolve** — objeto do comando, **sem envelope**:

```
função(data, authToken, tenantIds):
    se !data.razaosocial: lança erro "razaosocial é obrigatória"
    retorna { ...data, razaosocial: data.razaosocial.trim() }
```

```json
{ "razaosocial": "Acme Indústria Ltda", "cnpj": "00.000.000/0001-00",
  "email": "contato@exemplo.invalid", "status": "criada" }
```

> ⚠️ **Devolver uma chave que não está entre os atributos do comando não faz nada.** Se este processador
> quisesse acrescentar `cnpjvalidado`, o campo precisaria **antes** existir como atributo de `criar` no
> modelo — devolvido sem isso, é descartado em silêncio e não chega ao evento.

## 2 · Coordenação (saga) — `acme.financeiro` → projeto

**Onde o gancho está declarado** — mesmo modelo, evento `criada` do agregado `financeiro.conta`:

```json
"criada": {
  "domainBus": {
    "triggerCoordination": [
      { "name": "projeto_from_financeiro_criada",
        "targetTenantId": "00000000-0000-4000-8000-000000000005",
        "br": { "route": "acme/financeiro/financeiro/conta/coordination/projeto_from_financeiro_criada" } }
    ]
  }
}
```

**O que chega no processador** — um argumento só, sem token, sem cabeçalho:

```json
{
  "route": "acme/financeiro/financeiro/conta/coordination/projeto_from_financeiro_criada",
  "data": {
    "conta": {
      "aggregateid": "<uuid da conta>",
      "razaosocial": "Acme Indústria Ltda",
      "cnpj": "00.000.000/0001-00",
      "status": "criada",
      "criadaem": "2026-01-31 14:05:00"
    },
    "aggregateState": { "…estado reconstituído da conta…" },
    "targetTenantId": "00000000-0000-4000-8000-000000000005",
    "sourceTenantId": "00000000-0000-4000-8000-000000000002"
  }
}
```

A primeira chave de `data` é o **tipo do agregado de origem** (`conta`), e `aggregateid` já vem incluído.
`aggregateState` **pode vir `{}`** sem aviso — trate como normal.

**O que o processador devolve** — com envelope:

```
função(data):
    origem = data.conta
    retorna { processedData: { targetCommand: {
        boundedContext: "engenharia",
        aggregateType:  "projeto",
        commandName:    "criar",
        data: { nome: origem.razaosocial, contaid: origem.aggregateid, status: "criado" }
    } } }
```

O persistence-crs submete esse `targetCommand` como **novo comando** no contexto destino (CP-7).

> ⚠️ **Um comando, nunca uma lista** — e devolver **sem** o envelope `processedData` responde `200` e
> descarta a saga em silêncio. Ver [contextos.md](contextos.md).

## 3 · Projeção cross-contexto — `acme.engenharia`

**Onde o gancho está declarado** — [`examples/acme.engenharia.model.json`](../examples/README.md),
evento `recusada` do agregado `engenharia.demanda`:

```json
"recusada": {
  "whenAttribute": "recusadaem",
  "domainBus": {
    "triggerProjection": [
      { "name": "demandadecisao",
        "targetTenantId": "00000000-0000-4000-8000-000000000005",
        "targetSchema": "engenharia",
        "br": { "route": "acme/engenharia/engenharia/demanda/projection/demandadecisao_from_demanda" } }
    ]
  }
}
```

> A rota declarada no modelo e o caminho do arquivo publicado precisam ser **idênticos entre si**; a
> [forma canônica](README.md#forma-canônica-da-rota-obrigatória) é o que evita colisão entre organizações.
> O modelo de exemplo no acervo declara essa rota **sem o segmento do agregado** — aqui ela aparece na
> forma canônica.

**O que chega no processador** — um argumento, e o contexto aninhado em `_meta`:

```json
{
  "route": "acme/engenharia/engenharia/demanda/projection/demandadecisao_from_demanda",
  "data": {
    "demanda": {
      "aggregateid": "<uuid da demanda>",
      "parecer": "Fora do escopo contratado.",
      "motivorecusa": "escopo",
      "status": "recusada",
      "recusadaem": "2026-01-31 14:05:00"
    },
    "_meta": {
      "targetTenantId": "00000000-0000-4000-8000-000000000005",
      "sourceTenantId": "00000000-0000-4000-8000-000000000005",
      "targetSchema": "engenharia"
    }
  }
}
```

**O que o processador devolve** — envelope, e a chave é o nome da **projeção de destino**:

```
função(data):
    origem = data.demanda
    retorna { processedData: { demandadecisao: {
        aggregateid:  origem.aggregateid,     # obrigatório — decide criar ou atualizar
        decisao:      "recusada",
        motivo:       origem.motivorecusa,
        registradaem: origem.recusadaem
    } } }
```

Existindo linha com aquele `aggregateid`, o retorno é aplicado sobre ela; não existindo, uma linha nova é
criada. **Sem `aggregateid`, o consumo falha.**

## Rota desconhecida

```
POST /br
{ "route": "acme/rota/que/nao/existe", "data": {} }

→ 400 { "status": "error", "mensagem": "Invalid route ... Available routes: ...", "tipo": "Error" }
```

Quando a organização ainda não publicou nenhum processador, a lista vem vazia e a mensagem diz isso, em
vez de listar rotas que não são publicáveis.

## Corpo inválido (formato de erro diferente)

```
POST /br
{ "data": { "razaosocial": "Acme" } }      # sem "route"

→ 400 { "erro": "Validacao falhou." }      # um campo só — sem status/mensagem/tipo
```

> No contexto síncrono, esse `400` **não** é o que o cliente final recebe: ele vê `510`. Ver
> [erros.md](erros.md).
