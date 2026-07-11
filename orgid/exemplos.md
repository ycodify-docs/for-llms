# orgid · exemplos anotados

> Dados genéricos. `/up`/`/ua` levam `Authorization`; `/open/*` é público. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto.

## 1. Começar um cliente (registro público — antes do forger)

```
POST /open/up/account
{ "account": { "username": "alice", "email": "alice@acme.com", "password": "•••" },
  "org": { "name": "acme", "client": "AcmeLtda", "alias": "ACME" } }
→ 201   // cria conta (pendente) + org "acme" + vínculo administrador
// "client" = nome da empresa cliente (obrigatório, ≤12 caracteres)
```

Depois: ativar a conta (fluxo de hash) e, então, usar o forger sob `/org/acme/...`.

## 2. Ativar/recuperar senha (hash)

```
GET /open/up/hash/for/recuperar/by/alice        → 200 {}     // envia hash de vida curta
PUT /open/up/account/alice/recuperar/hash/<hash>
{ "password": "•••" }                            → 200
```

## 3. Ler a própria conta (autenticado)

```
GET /up/account/by/jwt
Headers: Authorization: ...
→ 200 { "username": "alice", "name": "Alice", "email": "...", "status": "ACTIVE" }   // sem senha
```

## 4. Criar organização (administrador)

```
POST /up/org
Headers: Authorization: ...
{ "name": "acme", "client": "AcmeLtda", "alias": "ACME" }   // client obrigatório (≤12)
→ 200 { "id": 30, "name": "acme", "client": "AcmeLtda", ... }
```

## 5. Vincular conta a papel numa org (multi-tenant)

```
POST /up/account-role-org
Headers: Authorization: ...
{ "account": { "username": "bob" }, "role": { "name": "API_ENGINEER" }, "org": { "name": "acme" } }
→ 201
```

## 6. Papel externo público

```
GET /ua/open/role/owner/acme        → 200 [ { "name": "cliente", "owner": "acme", "ispublic": true } ]
```

## 7. Associar papel externo a uma conta (`/ua` — note o `owner`)

Diferente do #5 (`/up`, papel `API_*` **sem** `owner`): no `/ua` o papel tem **dono** (`owner`),
**obrigatório** no corpo. Aceita objeto único ou array.

```
POST /ua/account-role/using/authority
Headers: Authorization: ...
{ "account": { "username": "bob" }, "role": { "name": "cliente", "owner": "acme" } }
→ 200
```

> Conta e papel (`name`+`owner`) devem **existir**. Corpo incompleto / contexto que impeça montar a
> associação → `510` (falha não tratada, não validação de campo). Ver [erros.md](erros.md) e
> [endpoints/ua-associacao.md](endpoints/ua-associacao.md).

> O `name` da org (`acme`) é o `{org}` dos caminhos do forger. Papéis `/up` são `API_*`; papéis `/ua`
> têm `owner` (espaço de nomes). Ver [README](README.md) e [coordenação](../coordenacao.md).
