# shell · BFF — a página mudou de lugar

> **O contrato do BFF está em [`bff/README.md`](../bff/README.md).** Esta página era uma cópia dele
> dentro da seção `shell/`, e as duas divergiram: a de lá é a viva.

Por que o BFF não fica sob `shell/`: ele **não é parte da casca**. Serve à casca **e** a cliente sem UI
(app móvel, integração, serviço), então é serviço **irmão** — ver [README](README.md).

O que está lá e **não existia** nesta cópia:

| Assunto | Seção em `bff/README.md` |
|---|---|
| **Quem autoriza a leitura** — `query` não aplica portão de papel; decide o `accessControl.read` da entity | "Quem autoriza a leitura" |
| **Value object no comando** — `single` objeto, `multiple` array de objetos, escalar recusado | "Proxy de domínio" |
| **`paging` é obrigatório na prática** — sem ele a lista vem cortada e ninguém avisa | "Proxy de domínio" |
| **`truncated`** — a resposta passa a dizer quando cortou | "Proxy de domínio" |
| **`401` com `reason`** — `no-session` · `expired` · `evicted` · `unknown` | "`401` diz por que não há sessão" |
| Sessão, capacidade, resolução do miolo, autocadastro, erros | o documento inteiro |

Esta página continua existindo como ponteiro porque outras da seção `shell/` apontam para ela. Ao
editar qualquer uma delas, troque o link por `../bff/README.md`.
