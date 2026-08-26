# bff — guia do serviço (endpoints e contratos)

> O **BFF** (Backend-for-Frontend) é a **borda de consumo** da plataforma: faz o login, guarda o token
> (cookie httpOnly), **deriva a capacidade** do tenant, **resolve o miolo** por tenant e faz **proxy** ao
> gateway injetando as credenciais **no servidor**. Quem consome fala **só** com o BFF — nenhum segredo
> sai para o cliente.
>
> **Não é exclusivo de frontend.** A casca universal é o consumidor mais comum, mas o BFF atende
> igualmente **cliente sem UI** (sistema backend/API-only, app móvel, integração de terceiro): a
> resolução de miolo simplesmente não é usada, e sessão/capacidade/proxy valem igual. Por isso vive aqui
> como **serviço próprio**, e não sob `shell/`.
>
> Pré: [06-autenticacao](../06-autenticacao.md). Para o uso com casca+miolo:
> [shell/README](../shell/README.md), [shell/seguranca](../shell/seguranca.md).
>
> _(Contratos abaixo. Os **endereços** dos serviços da plataforma são **configuração de deploy** — não
> constam aqui; ver operacional em `yc.app/docs`.)_

## Sessão

| Operação | Método · Path | Corpo / Query | Resposta |
|---|---|---|---|
| Login | `POST /session/login` | `{ username, password }` | identidade + **árvore de tenants** + papéis; **cookie httpOnly** (token). **Sem token no corpo.** |
| Sessão atual | `GET /session/me` | — | mesma projeção segura, ou `401` sem sessão |
| Encerrar | `POST /session/logout` | — | limpa o cookie |

`POST /session/login` chama o [auth](../auth/README.md) (`/ua/sign-in`), guarda o token e devolve **só a
projeção segura**:

```jsonc
{
  "identity":   { "username": "...", "name": "...", "email": "...", "status": "...", "exp": 0 },
  "orgs":       [ { "owner": "...", "name": "...", "alias": "...", "status": "..." } ],
  "roles":      [ { "role": "...", "org": "...", "status": "...", "alias": "..." } ],
  "tenants":    [ /* tuplas org:project:boundedContext:dataschema:tenant */ ],
  "tenantTree": [ { "org": "...", "projects": [ { "project": "...", "boundedContexts": [ { "boundedContext": "...", "tenantId": "..." } ] } ] } ]
}
```

## Capacidade

| Operação | Método · Path | Resposta |
|---|---|---|
| Capacidade do tenant | `GET /session/capabilities?tenant={tenantId}` | **modelo de capacidade** (ver [contrato-miolo](../shell/contrato-miolo.md#modelo-de-capacidade)) |

O BFF lê o **modelo publicado** do tenant no [forger](../forger/endpoints/model.md) e cruza com os
**papéis org-scoped** do token → devolve **só** os comandos autorizados. Só para tenant que **consta no
token**; senão `404`.

## Miolo

| Operação | Método · Path | Resposta |
|---|---|---|
| Manifesto do miolo | `GET /tenant/{tenantId}/miolo-manifest` | `{ "tenantId": "...", "manifestUrl": "..." }` |

Resolve `tenant → URL` do miolo (ver [injecao](../shell/injecao.md)). Só para tenant do usuário.

## Proxy de domínio

O BFF expõe o caminho de **comando** e **consulta** ao miolo (via `api` do hostContext), fazendo **proxy**
ao [persistence-crs](../persistence-crs/README.md) (escrita) e ao [persistence-q](../persistence-q/README.md)
(leitura) — **injetando `Authorization` + `X-Tenant-Id` no servidor**. O miolo **não** compõe esses
cabeçalhos nem conhece o token.

## Preferências da organização (org-scoped)

Preferências de **apresentação por organização** — valem para **todos** os usuários da org, persistidas no
BFF. Leitura por qualquer membro **ativo**; escrita **só por MASTER-na-org** (revalidada no servidor).

| Operação | Método · Path | Corpo / Query | Resposta |
|---|---|---|---|
| Ler prefs | `GET /session/org-preferences?org={org}` | — | `{ org, prefs, canConfigure }` |
| Alterar prefs | `PUT /session/org-preferences` | `{ org, formMode?, colorScheme? }` (**patch parcial**) | `{ org, prefs, canConfigure: true }` |

```jsonc
// prefs
{ "formMode": "inline" | "modal",           // layout do form de comando no miolo
  "colorScheme": "verde" | "azul" | "ambar" | "roxo" | "cinza" }  // esquema de cor da casca (ver ../shell/estilo.md)
```

- **Patch parcial**: o `PUT` aceita `formMode` e/ou `colorScheme`; campos ausentes mantêm o valor atual.
  Valores fora do catálogo → `400`. `canConfigure` = o portador é MASTER **naquela** org.
- O catálogo de `colorScheme` é **contrato** — espelhado no shell (catálogo/UI) e no BFF (validação).
  Ver [estilo](../shell/estilo.md#esquemas-de-cor-nomeados-paleta-completa-por-organização).

## Erros

| HTTP | Quando |
|---|---|
| `400` | falta parâmetro (ex.: `username`/`password`, `tenant`) |
| `401` | sem sessão / sessão inválida / credencial inválida |
| `404` | tenant não pertence ao usuário / miolo não registrado |
| `502` | falha ao falar com um serviço da plataforma |

## Checklist do agente

- [ ] O browser fala **só** com o BFF — nunca com a plataforma direto.
- [ ] O token **nunca** vai no corpo — só em **cookie httpOnly**.
- [ ] `Authorization`/`X-Tenant-Id` são compostos **no BFF** (ver [seguranca](../shell/seguranca.md)).
- [ ] Endereços de serviço = **config de deploy**, nunca hardcode nem em doc pública.
