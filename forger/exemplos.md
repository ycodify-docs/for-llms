# forger · exemplos anotados

> Sequência mínima para implantar um sistema do zero. Dados genéricos (`acme`, `vendas`).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.
> Guia: [README.md](README.md). Ordem completa: [fluxo de deploy](../03-fluxo-de-deploy.md).

> Todas as requisições levam `Authorization`. As respostas mostram só os campos relevantes.

## 1. Conexão (dbconn)

```
POST /org/acme/dbconn
{ "dbsqlserveraddress": "db.acme.internal", "dbsqlname": "principal",
  "dbsqlserverport": 5555, "dbsqlusername": "app", "dbsqluserpassword": "•••" }
→ 201 { "id": 10, "message": "..." }
```

## 2. Banco de leitura (database)

```
POST /org/acme/dbconn/10/database
{ "dbsqlname": "leitura" }
→ 201 { "id": 20, "dbsqlname": "leitura", "dbsqlusername": "...", "message": "..." }
```

## 3. Contexto (project)

```
POST /org/acme/project
{ "name": "vendas", "alias": "Vendas", "envtype": "prod" }
→ 201 { "id": 30, "tenantSecret": "<uuid>", "tenantPid": "<uuid>", "message": "..." }
```

## 4. Esquema (dataschema) — gera o tenant-id

```
POST /org/acme/project/vendas/database/20/dataschema
{ "name": "vendas" }
→ 201 { "id": 40, "message": "..." }   // tenant-id (UUID) fica associado ao esquema
```

## 5. Projeção (entity)

```
POST /org/acme/project/vendas/dataschema/vendas/entity/validate   // dry-run
{ "name": "pedido", "attributes": [ { "name": "total", "type": "decimal" } ] }
→ 200

POST /org/acme/project/vendas/dataschema/vendas/entity            // aplica CREATE TABLE
{ "name": "pedido", "attributes": [ { "name": "total", "type": "decimal" } ] }
→ 201
```

## 6. Modelo de domínio (model)

```
POST /org/acme/project/vendas/tenant/<tenant-id>/model
(upload do documento .json — ver examples/)
→ 201 { "key": "<referência de publicação>" }
```

Após o passo 6, o sistema aceita **comandos** (persistence-crs) e **consultas** (persistence-q).
Ver [arquitetura](../01-arquitetura.md) e os exemplos de modelo em [examples/](../examples/README.md).
