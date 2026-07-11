# auth · catálogo de erros

> Envelope e códigos. Guia: [README.md](README.md).

## Envelope
Em erro, a resposta tem o **status HTTP** e um corpo com `status` + mensagem.

## Códigos HTTP

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `200` | OK | Login/renovação bem-sucedidos. | — |
| `401` | Não autenticado | Credenciais inválidas (login); token ausente/inválido/expirado (renovação). | Conferir usuário/senha; renovar/refazer login. |
| `403` | Proibido | Conta **pendente** (em validação) — apenas `/up/sign-in`. | Ativar a conta antes (ver [orgid/registro](../orgid/endpoints/publico.md)). |
| `417` | Falha de dependência | Falha ao consultar identidade/descoberta de tenants/sessão. | Retentar; se persistir, escalar. |

## Notas
- O login (`sign-in`) é **público** (sem token). A **renovação** exige `Authorization`.
- A senha **nunca** é devolvida. O token é de **curta duração** — trate `401` por expiração com renovação.
- **Sessão única:** um login novo invalida sessões anteriores do mesmo usuário.
