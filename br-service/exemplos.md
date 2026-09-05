# br-service · exemplos anotados

> O br-service é chamado pelo persistence-crs. Os exemplos mostram o **contrato** e a **forma de um
> processador**. Dados genéricos. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.

## Chamada (como o persistence-crs invoca)

```
POST /br
{ "route": "acme/vendas/vendas/pedido/validar", "data": { "total": 250.0, "cliente": "C-1" } }

→ 200
{ "total": 250.0, "cliente": "C-1", "descontoAplicado": 0.0, "valido": true }
```

A resposta é o objeto **direto** do processador; o persistence-crs a usa para enriquecer/validar o comando.

As três categorias diferem pela **rota** (segmento de path) e pelo que retornam
(ver [README — três categorias](README.md#três-categorias-de-processador)).

> **Não há envelope.** A resposta é o objeto que o processador devolveu, tal e qual — como no exemplo
> acima. Um processador de **regra de negócio** devolve os campos do próprio comando: são as chaves desse
> objeto que o motor de comandos aproveita, e só as que já existiam no comando original. Devolver os dados
> aninhados dentro de outra chave faz o motor não encontrar nada para aproveitar.

### 1) Regra de negócio — rota `…/<agregado>/<comando>`

Valida/enriquece os dados do comando; o objeto devolvido volta para o comando antes do evento.

```
função(data):                                  # data = dados do comando
    se !data.cliente: lança erro "cliente obrigatório"
    retorna { ...data, cliente: data.cliente.trim() }    # direto, sem envelope
```

### 2) Coordenação (saga) — rota `…/<agregado>/coordination/<alvo>_from_<origem>`

Recebe o estado do agregado de origem; monta um **`targetCommand`** para o contexto destino.

```
função(data):                                  # data = estado do agregado de origem
    retorna { processedData: { targetCommand: {
        boundedContext: "financeiro",
        aggregateType:  "conta",
        commandName:    "criar",
        data: { razaosocial: data.razaosocial, cnpj: data.cnpj }
    } } }
```

O persistence-crs submete esse `targetCommand` como **novo comando** (CP-7).

### 3) Projeção cross-contexto — rota `…/<agregado>/projection/<alvo>_from_<origem>`

Mapeia o evento de origem na **linha da projeção** de outro contexto; pode **ler** a projeção destino
(somente leitura) para um UPDATE coerente.

```
função(data):                                  # data = estado do agregado de origem
    retorna { processedData: { "demandadecisao": {
        aggregateid: data.demandaid,            # chave da linha na projeção destino
        decisao:     data.decisao,
        registradaem: data.registradaem
    } } }
```

## Rota desconhecida

```
POST /br
{ "route": "acme/rota/que/nao/existe", "data": {} }

→ 400 { "status": "error", "mensagem": "Invalid route ... Available routes: ...", "tipo": "Error" }
```

Quando a organização ainda não publicou nenhum processador, a lista vem vazia e a mensagem diz isso, em vez
de listar rotas que não são publicáveis.

## Corpo inválido (formato de erro diferente)

```
POST /br
{ "data": { "total": 250.0 } }            # sem "route"

→ 400 { "erro": "Validacao falhou." }     # um campo só — sem status/mensagem/tipo
```

> A rota referenciada deve coincidir com a declarada no **model** do comando/evento (ver
> [examples/](../examples/README.md) e [forger/model](../forger/endpoints/model.md)).
