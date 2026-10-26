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

Cabeçalhos: `Content-Type: application/json`. **Não** exige `Authorization` (é o login). Pela borda, exige
**`X-Forger-Credential`** — é a **única** credencial da rota de auth, e não há alternativa. Com `X-Tenant-Id` no
lugar dele, o login responde `401` de corpo vazio; sem nenhum dos dois, `401` `missing_headers`. Ver
[gateway](../../gateway/README.md).

## Resposta

`200`:

```jsonc
{ "accessToken": "<token de acesso>", "tokenType": "Bearer",
  "id": 0, "username": "...", "name": "...", "email": "...",
  "roles": ["..."], "mcpSessionId": "<uuid da sessão>" }
```

Use `accessToken` em `Authorization: Bearer <token>` nas chamadas seguintes — a chave é `accessToken`, e não
`token`: quem lê `.token` fica sem token. O token embute usuário, papéis,
organizações e tenants. Ver [README — claims](../README.md#o-que-o-login-retorna-token--claims).

## Token ES256 e chave pública (JWKS)

> **Em `develop` (auth `ca02e61`, composer `aaad2c2`), ainda não no ar em 2026-10-26.** Liga quando a chave for
> configurada e implantada (pedido à infra). Até lá, o campo não vem na resposta e o JWKS responde `{"keys":[]}`.

Para o serviço que precisa validar o token **sem guardar o segredo compartilhado**, o login (`/up` e `/ua`) e a
[renovação](renovar.md) devolvem também:

| Campo | Tipo | Significado |
|---|---|---|
| `accessTokenEs256` | string | o mesmo token de acesso, com os mesmos claims, o mesmo emissor (`iss` = `YC-AuthService`), o mesmo `sub` e a mesma validade, assinado em **ES256** (ECDSA P-256) com `kid` no cabeçalho |

- **Chave pública:** `GET /ua/.well-known/jwks.json`, público → `200 {"keys":[{"kty":"EC","crv":"P-256","use":"sig",
  "alg":"ES256","kid":"…","x":"…","y":"…"}]}`. Escolha a chave pelo `kid` do cabeçalho do token.
- **Para validar:** confira a assinatura ES256 com a chave daquele `kid`, depois `iss` = `YC-AuthService` e `exp`.
- **O `accessToken` não muda.** É ele que vai em `Authorization` para os serviços da plataforma. O
  `accessTokenEs256` serve só a quem valida pela chave pública.

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

## Só em `/ua`: captcha e aceite de termos

> **Em vigor desde 2026-10-16T21:06:53Z.** Medido pela borda às 21:29:18Z: conta de org com manifesto, sem
> captcha → `428 {"codigo":"captcha_ausente"}`.

Para conta que tem papel em organização que publicou o [manifesto da aplicação](../../orgid/endpoints/aplicacao.md):

| Campo novo | Tipo | Significado |
|---|---|---|
| `captcha` | string | a solução do desafio ALTCHA, em base64: `{algorithm, challenge, number, salt, signature}` |
| `domain` | string | o host por onde a pessoa chegou (`location.hostname`) |
| `aceite` | array | `[{"org": "<org>", "termos": <versão>, "privacidade": <versão>}]` — as versões aceitas |

- **Desafio:** `GET /ua/sign-in/challenge`, público → `200 {algorithm:"SHA-256", challenge, maxnumber, salt,
  signature}`. Vale 5 minutos e serve **uma vez** (no login ou no autocadastro).
- **Ordem:** o captcha é conferido **antes** da senha; a senha; o aceite **depois** da senha.
- **Org exigida:** a do `domain`, se ele for domínio verificado de uma org da conta; senão, **todas** as orgs da
  conta que publicaram termos.

| Situação | Resposta |
|---|---|
| captcha exigido e ausente | `428 {"codigo":"captcha_ausente"}` |
| captcha inválido, vencido ou já usado | `428 {"codigo":"captcha_invalido"}` — peça outro desafio |
| aceite faltando ou de versão que não é a vigente | `428 {"codigo":"aceite_pendente","pendentes":[{org, termos:{versao, vigenteDesde, texto}\|null, privacidade:{…}\|null}]}` — mostre os textos e reenvie com `aceite` |
| tudo certo | `200`, a resposta de sempre; o aceite fica gravado |

O aceite é **uma vez por versão**: quem já aceitou a versão vigente não vê `aceite_pendente`.

## Comportamento
Segue o [fluxo de login](../README.md#fluxo-de-login-ciclo-de-vida): valida (via identidade) → descobre
contexto/tenants → emite token → registra sessão (login novo **invalida** sessões anteriores do usuário).
