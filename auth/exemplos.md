# auth · exemplos anotados

> Dados genéricos. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não literais.

## 1. Login na plataforma

```
POST /up/sign-in
{ "username": "alice", "password": "•••" }
→ 200
{ "accessToken": "<token>", "tokenType": "Bearer", "id": 1, "username": "alice",
  "name": "Alice", "email": "alice@acme.com", "roles": ["API_MASTER"], "mcpSessionId": "<uuid>" }
```

Use o token nas chamadas seguintes: `Authorization: Bearer <token>`.

## 2. Login externo

```
POST /ua/sign-in
{ "username": "cliente1", "password": "•••" }
→ 200  { "accessToken": "<token>", "tokenType": "Bearer", ... }   // accessToken inclui os tenants do usuário
```

## 3. Usar o token (em outro serviço)

```
POST /a/vendas/pedido
Headers: Authorization: Bearer <token>, X-Tenant-Id: <tenant-id>
{ "criar": { ... } }
```

## 4. Renovar

```
GET /up/sign-in/renew
Headers: Authorization: Bearer <token atual>
→ 200  { "accessToken": "<novo token>", ... }
```

## 5. Erros comuns

```
POST /up/sign-in   { "username": "alice", "password": "errada" }   → 401  (credencial inválida)
POST /up/sign-in   (conta ainda em validação)                      → 403  (pendente)
GET  /up/sign-in/renew   (sem token ou token expirado)             → 401
```

> Sequência típica: **auth** (login → token) → usar o token nos serviços (com `X-Tenant-Id` quando for
> dado por tenant). Como obter o `X-Tenant-Id`: [06-autenticacao.md](../06-autenticacao.md).
