# Walkthrough ponta a ponta — do zero à consulta

> Jornada **canônica e única** para construir um sistema do nada até consultar dados, reunindo todos os
> serviços numa sequência só. Use o exemplo [`examples/acme.vendas.model.json`](examples/acme.vendas.model.json)
> como modelo. Pré-requisitos: [conceitos](02-conceitos.md), [arquitetura](01-arquitetura.md).
>
> Os blocos `{ ... }` abaixo são **ilustrativos** (com placeholders `<...>`), não JSON literal pronto.
> Todas as requisições autenticadas levam `Authorization`; comandos/consultas levam também `X-Tenant-Id`.
> Como obter esses dois: [06-autenticacao.md](06-autenticacao.md).

## Contents
- Visão da jornada
- Passo 1–6 · Implantar recursos (forger)
- Passo 7 · Publicar o modelo (forger)
- Passo 8 · Executar comando (persistence-crs)
- Passo 9 · Projeção assíncrona (es-n)
- Passo 10 · Consultar (persistence-q)
- Checklist

---

## Visão da jornada

```
forger:  dbconn → database → project → dataschema → entity → (publicar model)
                                                                     │
persistence-crs:  POST /a/...  (comando)  ── grava evento ──▶ es-n ──▶ projeção
                                                                     │
persistence-q:  POST /  (consulta a projeção)  ◀─────────────────────┘
```

## Passos 1–6 · Implantar recursos (forger)

A ordem é obrigatória ([fluxo de deploy](03-fluxo-de-deploy.md)). Resumo:

```
1. POST /org/acme/dbconn                                   → { id: 10 }
2. POST /org/acme/dbconn/10/database                       → { id: 20 }
3. POST /org/acme/project                                  → { id: 30, tenantPid, tenantSecret }
4. POST /org/acme/project/vendas/database/20/dataschema    → { id: 40 }   // gera o tenant-id
5. POST .../dataschema/vendas/entity/validate  e  .../entity   // cria a projeção (tabela de leitura)
```

Detalhe de cada um: [forger](forger/README.md). Regras de nome/limites e tipos de atributo:
[forger/README — regras de nome](forger/README.md#regras-de-nome-dos-recursos) e
[model-format — tipos](persistence-crs/spec/model-format.md#atributos-e-tipos).

> O **`tenant-id`** nasce no passo 4. Recupere-o quando precisar (ver [06-autenticacao.md](06-autenticacao.md)).

## Passo 7 · Publicar o modelo (forger)

```
POST /org/acme/project/vendas/tenant/<tenant-id>/model
(corpo = o documento .model.json — ver examples/acme.vendas.model.json)
→ 201 { "key": "<referência de publicação>" }
```

A partir daqui, persistence-crs/es-n/persistence-q **leem** esse modelo (CP-1). Gramática do documento:
[model-format.md](persistence-crs/spec/model-format.md).

> Publicar o modelo **não cria filas nem tabelas** — as tabelas vêm da `entity` (passo 5); as filas de
> projeção/coordenação são infraestrutura fixa do es-n. Ver
> [arquitetura — de onde vêm as filas](01-arquitetura.md#origem-das-filas).

## Passo 7b · Ativar o dataschema (`MODELING → RUNNING`)

Os passos 5–7 (entity/model) rodam com o dataschema em **`MODELING`** (esquema em edição). **Antes de
executar comandos/consultas**, transite o dataschema para **`RUNNING`**:

```
PUT /org/acme/project/vendas/database/20/dataschema/vendas
{ "logversion": <atual>, "status": "RUNNING" }   → 200
```

Em `MODELING`, persistence-crs/persistence-q **não interpretam** o modelo (o passo 8 falharia). Para
**editar o schema** depois, volte a `MODELING`, altere, e retorne a `RUNNING`. Ver
[forger/dataschema — gate de status](forger/endpoints/dataschema.md#atualizar).

## Passo 8 · Executar comando (persistence-crs)

```
POST /a/vendas/pedido
Headers: Authorization, X-Tenant-Id: <tenant-id>
{ "criar": { "cliente": "C-1", "total": 25000 } }       // total em centavos (tipo Long)
→ 200 { "id": "<uuid do agregado>", "status": "criada" }
```

O serviço valida a transição (`fromState`/`endState`), chama o **br-service** se o comando declarar
`br.route`, grava o evento e **notifica o es-n** (CP-3/CP-6). Ver
[crs/comando](persistence-crs/endpoints/comando.md).

## Passo 9 · Projeção assíncrona (es-n)

A gravação do evento notifica o **es-n**, que reconstitui o agregado e o despacha; o consumidor aplica a
**projeção** no banco de leitura (CP-4/CP-5). **É assíncrono:** logo após o passo 8, a projeção pode
ainda não refletir a mudança. Ver [es-n](es-n/README.md).

## Passo 10 · Consultar (persistence-q)

```
POST /
Headers: Authorization, X-Tenant-Id: <tenant-id>
{ "pedidosCriados": { "status": "criada" } }
→ 200 [ { "pedidosCriados": [ { "id": "<uuid>", "status": "criada", "total": 25000 } ] } ]
```

Tolere a breve defasagem de propagação (reconsulte se necessário). Ver
[persistence-q/consulta](persistence-q/endpoints/consulta.md).

## Checklist

```
[ ] 1–4  recursos criados na ordem (dbconn→database→project→dataschema); tenant-id obtido
[ ] 5    entity(s) criadas — a projeção (tabela de leitura) existe ANTES do 1º comando
[ ] 6/7  modelo .model.json válido (ver model-format) e publicado
[ ] 7b   dataschema ativado (status RUNNING) — SEM isso o comando do passo 8 falha
[ ] 8    comando aceito (200) — transição válida; status retornado
[ ] 9    aguardar propagação assíncrona da projeção
[ ] 10   consulta retorna o estado esperado
```
