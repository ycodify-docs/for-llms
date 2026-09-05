# auth — guia do serviço

> **Papel:** **emissão de token (login)** — autentica credenciais e devolve um **token de acesso** com
> as informações de identidade (usuário, papéis, organizações, tenants). É a **porta de entrada de
> identidade**: obtenha aqui o token e use-o em `Authorization` nos demais serviços. Forma o IAM da
> plataforma junto com o [orgid](../orgid/README.md) (auth **emite**; orgid **administra**; todos os
> demais **validam**). Pré-requisitos: [conceitos](../02-conceitos.md), [autenticação](../06-autenticacao.md).

## Contents
- Papel e o ciclo emite→valida
- Dois domínios (`/up` e `/ua`)
- Índice de endpoints
- O que o login retorna (token + claims)
- Fluxo de login (ciclo de vida)
- Pontos de coordenação
- Pitfalls (checklist do agente)

---

## Papel e o ciclo emite→valida

```
cliente ── credenciais ──▶ auth  ── valida (orgid) ──▶ emite TOKEN ──▶ cliente
                                                                  │
   cliente usa Authorization: Bearer <token> ──▶ forger / crs / q / br / filer / orgid
                                                  (cada um VALIDA o token; auth não valida)
```

auth **emite**; **não valida** token (validação é de cada serviço consumidor). O token carrega a
identidade completa para que os demais autorizem **sem** reconsultar a cada chamada.

## Dois domínios (`/up` e `/ua`)

| Prefixo | Domínio | Quem autentica |
|---|---|---|
| `/up` | **plataforma** | contas/parceiros que operam a plataforma (papéis `API_*`). |
| `/ua` | **externo** | usuários finais dos sistemas construídos sobre a plataforma. |

Espelha os domínios do [orgid](../orgid/README.md). No `/ua`, o login também **descobre os tenants** do
usuário (via forger) e os inclui no token.

## Índice de endpoints

> Endpoints de **login** são públicos (sem token); a **renovação** exige o token atual. Endpoints
> internos/observabilidade não constam.

| Operação | Método · Path | Documento |
|---|---|---|
| Login (plataforma) | `POST /up/sign-in` | [endpoints/sign-in.md](endpoints/sign-in.md) |
| Login (externo) | `POST /ua/sign-in` | [endpoints/sign-in.md](endpoints/sign-in.md) |
| Renovar token | `GET /up/sign-in/renew` | [endpoints/renovar.md](endpoints/renovar.md) |

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md). Contrato de fio: [openapi.yaml](openapi.yaml).

## O que o login retorna (token + claims)

Resposta de sucesso (`200`):

```jsonc
{ "token": "<token de acesso>", "type": "Bearer",
  "id": 0, "username": "...", "name": "...", "email": "...",
  "roles": ["..."], "sessionId": "<uuid da sessão>" }
```

O **token de acesso** é o que vai em `Authorization: Bearer <token>`. Ele embute, como atributos
(claims): **usuário**, **papéis**, **organizações** e **tenants** (`tenantId`s + descritores). É de
**curta duração** — renove pelo endpoint de renovação. (A plataforma também usa um token de renovação
de duração maior, internamente.) Trate o token como **opaco**; não dependa de sua estrutura interna.

## Fluxo de login (ciclo de vida)

1. **Recebe** `{ username, password }` em `/up/sign-in` ou `/ua/sign-in`.
2. **Valida credenciais** consultando o serviço de identidade (orgid): existência, senha, status, papéis.
3. **Descobre contexto** — organizações/papéis; no `/ua`, também os **tenants** do usuário (via forger).
4. **Emite o token** com os claims de identidade; registra a **sessão** (login novo invalida sessões
   anteriores do mesmo usuário — sessão única).
5. **Responde** `200` com token + dados. Conta **pendente** (em validação) → `403`; credencial inválida → `401`.

## Pontos de coordenação

- **CP-12** — auth **emite** o token que **todos** os demais serviços **validam** (mesmo segredo
  compartilhado). É o início da cadeia de autorização.
- auth → **orgid** (validar credenciais, obter usuário/papéis/organizações).
- auth → **forger** (no `/ua`, descobrir os tenants do usuário para o token).
- auth → **cache** (registrar a sessão; sessão única por usuário).

Ver [coordenação](../coordenacao.md) e [autenticação](../06-autenticacao.md).

## Pitfalls (checklist do agente)

- [ ] **Autentique primeiro:** obtenha o token no auth **antes** de chamar qualquer endpoint autenticado.
- [ ] Escolha o domínio certo: `/up` (plataforma) vs `/ua` (externo).
- [ ] Conta **pendente** → `403` (ative a conta antes — ver [orgid/registro](../orgid/endpoints/publico.md)).
- [ ] Token é de **curta duração** — trate `401` por expiração renovando (`/up/sign-in/renew`).
- [ ] **Login novo invalida o anterior** (sessão única) — não mantenha dois tokens ativos do mesmo usuário.
- [ ] Não inspecione/derive do conteúdo do token — trate-o como opaco.
