# shell — guia da casca universal + BFF

> **Papel:** a **aplicação única e universal** — uma **casca (shell)** fixa e um **BFF** que servem
> **todos os clientes**. Após o login, a casca descobre pelo **token** (via BFF) as organizações, os
> **bounded contexts** (tenants) e os **comandos autorizados** do usuário, monta o menu e **injeta em
> runtime**, por tenant, o **miolo** — a UI específica daquele cliente. **Não é um serviço da
> plataforma**; é a **camada de aplicação** que consome os oito serviços (via BFF).
> Pré-requisitos: [conceitos](../02-conceitos.md), [autenticação](../06-autenticacao.md).

## Contents
- Papel e a fronteira casca ↔ miolo
- Camadas
- Fluxo (login → menu → injeção)
- Documentos da seção
- Invariantes

---

## Papel e a fronteira casca ↔ miolo

A casca é **igual para todos**; o que muda por cliente é o **miolo**.

- **Casca (shell)** — login, resolução de tenant, **descoberta de capacidade**, roteamento, menu de
  bounded contexts, barra superior (seletor de organização, tema, perfil) e o **host** que injeta o
  miolo. Construída **uma vez** (infra de plataforma).
- **Miolo** — os componentes de tela e sua disposição (tabela, formulário, cards, …) de **um** bounded
  context. Concebido/codificado **por cliente** e **injetado em runtime** (ver [injecao](injecao.md)).
- **Fronteira** — o **modelo de capacidade** (agregados → comandos → atributos, derivado do token × modelo
  do tenant) diz **o que pode**; o **miolo** diz **como se vê**. Contrato em [contrato-miolo](contrato-miolo.md).

```
browser ──▶ casca (app) ──▶ BFF ──▶ plataforma (gateway: auth/forger/crs/q/…)
              │                       (o BFF guarda o token; o browser não)
              ▼
          host injeta o MIOLO do tenant (remoto, por bounded context)
```

## Camadas

| # | Camada | Responsabilidade | Por-tenant? |
|---|---|---|---|
| L1 | **Casca** | login, tenant, descoberta de capacidade, menu por BC, barra, host do miolo | não |
| L2 | **Capacidade** (derivada) | `papéis-do-token-na-org × modelo` → comandos autorizados + esqueleto de formulário | dado varia |
| L3 | **Miolo** | apresentação (telas/widgets/estilo) de um bounded context | **sim** |
| L4 | **BFF** | custódia do token, login, derivação de capacidade, resolução tenant→miolo, proxy | serve todos |

## Fluxo (login → menu → injeção)

1. **Login** → o BFF autentica no [auth](../auth/README.md), guarda o token **no servidor** (não vai ao
   browser) e devolve a identidade + a **árvore de tenants** + os **papéis** (org-scoped) do token.
2. A casca monta o **menu**: **1 entrada por bounded context** (da árvore de tenants).
3. O usuário **seleciona um BC** → a casca pede ao BFF a **capacidade** e a **URL do miolo** do tenant.
4. A casca **injeta o miolo** e passa o **hostContext** (ver [contrato-miolo](contrato-miolo.md)).

## Documentos da seção

| Documento | Cobre |
|---|---|
| [contrato-miolo](contrato-miolo.md) | interface casca↔miolo: `MioloModule`, `hostContext`, modelo de capacidade |
| [injecao](injecao.md) | como/quando injetar; resolução tenant→remoto; **de onde vem o build** do miolo |
| [estilo](estilo.md) | guia-mestre de estilo: tokens semânticos, tema claro/escuro, **marca sobrescrevível** |
| [seguranca](seguranca.md) | segredo só no BFF; cookie; capacidade = UX; revalidação no servidor |
| [bff](bff.md) | endpoints do BFF (contratos) |

## Invariantes

- [ ] **Zero segredo no browser** — token e chaves vivem **só no BFF** (ver [seguranca](seguranca.md)).
- [ ] **Capacidade = UX** — a casca **esconde** o não-autorizado; o [persistence-crs](../persistence-crs/README.md)
      **revalida** cada comando no servidor. O gating da casca **nunca** é a fronteira de segurança.
- [ ] **1 entrada de menu por bounded context** — é a "solda" com o miolo.
- [ ] **A casca não muda por cliente** — só o **miolo** (e seus tokens de tema) varia.
- [ ] **Papéis são org-scoped** — a capacidade é calculada com os papéis **da organização** do tenant
      selecionado, nunca globalmente (ver [autenticação](../06-autenticacao.md)).
