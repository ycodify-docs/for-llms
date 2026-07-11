# auth · endpoints · renovar token

> Renova o token de acesso (emite um novo a partir do atual, recarregando a identidade). Guia: [../README.md](../README.md).

## Requisição
`GET /up/sign-in/renew`

Cabeçalhos: `Authorization: Bearer <token atual>` (**obrigatório** — diferente do login).
Sem corpo.

## Resposta
`200` — novo `JwtResponse` (mesma forma do [login](sign-in.md)): token renovado + dados de identidade.

## Comportamento
1. Valida o token atual (`Authorization`).
2. **Recarrega** a identidade do usuário (papéis/organizações atuais) — o token novo reflete o estado atual.
3. Emite o novo token e **registra nova sessão** (invalida as anteriores — sessão única).

> Use a renovação quando o token estiver perto de expirar ou após `401` por expiração. O token é de
> **curta duração**; a renovação tem duração própria, também curta.

## Erros
| HTTP | Quando |
|---|---|
| `401` | token ausente/inválido/expirado |
| `417` | falha de serviço dependente |

Catálogo: [../erros.md](../erros.md).
