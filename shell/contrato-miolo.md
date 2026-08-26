# shell · contrato casca ↔ miolo

> A **interface** entre a **casca (host)** e o **miolo (remoto)**. A casca carrega o miolo em runtime e
> o monta passando um **hostContext**; o miolo expõe um módulo conhecido — o **MioloModule**. O miolo
> **nunca** recebe token nem segredo — só o necessário para renderizar e para falar com o BFF.
> Pré: [README](README.md), [injecao](injecao.md).

## MioloModule — o que o miolo expõe

| Export | Assinatura | Papel |
|---|---|---|
| `mount` | `(el, hostContext) → dispose` | monta a UI do bounded context em `el`; devolve uma função de **desmontagem** |
| `getPresentationSchema?` | `() → PresentationSchema` | (opcional) esquema de apresentação que o miolo usa |
| `getDesignTokens?` | `() → Tokens` | (opcional) tokens de tema do miolo (ver [estilo](estilo.md)) |

O contrato **mínimo obrigatório** é `mount` (e o `dispose` que ele devolve). O resto é opcional.

## hostContext — o que a casca passa ao miolo

```jsonc
{
  "tenant":       { "org": "...", "project": "...", "boundedContext": "...", "tenantId": "..." },
  "capability":   { /* modelo de capacidade — abaixo */ },
  "identity":     { "username": "...", "name": "...", "email": "..." },
  "api":          { "command": "…", "query": "…", "aggregate": "…", "history": "…" },
  "prefs":        { "formMode": "inline" | "modal" },
  "canConfigure": false,
  "savePrefs":    "(formMode) => Promise<void>"
}
```

- **`tenant`** — o bounded context selecionado (org/projeto/BC + `tenantId` **opaco**).
- **`capability`** — o **modelo de capacidade** (abaixo): o que o usuário pode fazer ali.
- **`identity`** — dados **não-sensíveis** do usuário. **Sem token, sem senha.**
- **`api`** — as quatro operações de domínio, todas contra o **BFF** (nunca a plataforma direto):

  | Método | Para quê |
  |---|---|
  | `command({ tenantId, aggregate, command, data, id?, status? })` | dispara um comando (escrita) |
  | `query({ tenantId, aggregate, predicates? })` | consulta a projeção (leitura) |
  | `aggregate({ tenantId, aggregate, id })` | **estado autoritativo** do agregado |
  | `history({ tenantId, aggregate, id })` | histórico de eventos do agregado |

  O miolo **não** compõe `Authorization` nem `X-Tenant-Id` — o BFF injeta no servidor (ver
  [seguranca](seguranca.md), [bff](../bff/README.md)).

  > Use `aggregate()` — não `query()` — para carregar o estado de um agregado antes de uma transição:
  > a projeção é **assíncrona** e pode ainda não refletir o último comando. Depois de escrever,
  > **reconsulte** `query()` até o estado esperado aparecer (read-your-writes).

- **`prefs`** — preferência de apresentação **da organização** (hoje: `formMode`, o layout do form de
  comando). Vale para todos os usuários da org. Ver [bff](../bff/README.md).
- **`canConfigure`** — se **este** usuário pode alterar a preferência (é `MASTER` na org). É dica de
  UX: o BFF **revalida** no servidor.
- **`savePrefs(formMode)`** — persiste a preferência da org. Só tem efeito se o BFF confirmar o papel.

> **Ainda não providos:** `navigation` (navegação da casca) e `i18n` estiveram previstos neste
> contrato, mas **não existem** na implementação. Um miolo não deve contar com eles. Quando forem
> implementados, entram aqui **e** na casca na mesma mudança.

## Modelo de capacidade

Derivado de **(papéis do token na organização do tenant) × (modelo do tenant)**. Diz **o que pode**;
não diz **como se vê** (isso é o miolo).

```jsonc
{
  "org": "...", "project": "...", "tenantId": "...",
  "aggregates": [
    {
      "aggregate": "...", "boundedContext": "...",
      "commands": [
        {
          "name": "...",
          "roles": ["..."],
          "fromState": ["..."],
          "endState": "...",
          "attributes": [ { "name": "...", "type": "...", "role": "input|status|serverStamp|lookup" } ]
        }
      ]
    }
  ]
}
```

- **`commands`** — **apenas** os autorizados ao papel do usuário (interseção já aplicada pelo BFF).
- **`fromState`** — de quais estados o comando aparece; a casca/miolo habilita a transição só quando o
  agregado está num estado válido.
- **`attributes[].role`** — classificação que guia a apresentação:
  - `input` — campo editável comum.
  - `status` — campo de transição/concorrência (não é entrada do usuário).
  - `serverStamp` — carimbo do servidor (oculto no formulário).
  - `lookup` — referência a outro agregado (widget de seleção).

## Regras

- **Modelo = domínio (autoritativo)**: existência de comando, papéis, tipos, transições — o miolo **não
  sobrescreve**. **Miolo = apresentação**: rótulo, widget, layout, ordem, visibilidade — o modelo **não
  fornece**, o miolo **preenche**.
- **Nada de segredo no miolo**: toda escrita/leitura de domínio passa pelo `api` → **BFF** → plataforma,
  que **revalida** a autorização no servidor.
- **Ciclo de vida**: `mount` ao selecionar o bounded context; `dispose` ao trocar de BC ou encerrar.
