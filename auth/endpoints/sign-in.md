# auth · endpoints · login (sign-in)

> Autentica credenciais e devolve o **token de acesso**. Públicos (sem token). Guia: [../README.md](../README.md).

## Requisição

- Plataforma: `POST /up/sign-in`
- Externo: `POST /ua/sign-in`

Corpo (JSON):

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `username` | string | sim | Identificador do usuário. |
| `password` | string | sim | Senha. |

Cabeçalhos: `Content-Type: application/json`. **Não** exige `Authorization` (é o login).

## Resposta

`200`:

```jsonc
{ "accessToken": "<token de acesso>", "tokenType": "Bearer",
  "id": 0, "username": "...", "name": "...", "email": "...",
  "roles": ["..."], "mcpSessionId": "<uuid da sessão>" }
```

Use `token` em `Authorization: Bearer <token>` nas chamadas seguintes. O token embute usuário, papéis,
organizações e tenants. Ver [README — claims](../README.md#o-que-o-login-retorna-token--claims).

## Diferenças entre domínios
- **`/up`** (plataforma): valida e, se a conta estiver **pendente** (em validação), retorna `403`.
- **`/ua`** (externo): além de validar, **descobre os tenants** do usuário e os inclui no token.

## Erros
| HTTP | Quando |
|---|---|
| `401` | credenciais inválidas |
| `403` | conta **pendente** (em validação) — só `/up` |
| `417` | falha de serviço dependente (identidade/descoberta/sessão) |

Catálogo: [../erros.md](../erros.md).

## Comportamento
Segue o [fluxo de login](../README.md#fluxo-de-login-ciclo-de-vida): valida (via identidade) → descobre
contexto/tenants → emite token → registra sessão (login novo **invalida** sessões anteriores do usuário).
