# monitor — relatos de suporte (guia)

> **Papel:** guarda os **relatos de suporte** que os usuários da aplicação (casca + BFF) enviam e a
> **conversa** entre o usuário e a equipe da plataforma sobre cada relato. Quem grava e lê em nome do usuário
> é o **BFF**, com **credencial de serviço**. Quem atende é a **equipe** (`API_MASTER`/`API_ENGINEER`), pela
> tela do monitor. O relato **não** vira ticket em outro sistema.
> Pré-requisitos: [conceitos](../02-conceitos.md), [borda](../gateway/README.md), [BFF](../bff/README.md).

> **Estado em 2026-09-27:** o serviço está **implantado**, e a tela do monitor já tem a aba Suporte. A
> **rota na borda** (`/v3/monitor/suporte/**`) foi pedida e ainda não está publicada. Até lá, a borda recusa
> a chamada antes de ela chegar ao monitor
> ([gateway/erros.md](../gateway/erros.md#404--a-borda-não-reconheceu-a-rota)).

## Quem chama o quê

| Quem | Por onde | Como se identifica | O que alcança |
|---|---|---|---|
| **BFF**, em nome do usuário | `/v3/monitor/suporte/v1/…`, pela borda | credencial de serviço + cabeçalhos do relator | só os relatos **daquele usuário** |
| **Equipe da plataforma** | tela do monitor | token de acesso com `API_MASTER` ou `API_ENGINEER` | os relatos **da própria organização**; o operador vê todos |

**O browser nunca fala com o monitor.** O relator não se declara: o BFF tira da sessão, no servidor, o
usuário, a organização e o tenant, e os manda em cabeçalhos
([endpoints](endpoints/relatos.md#cabeçalhos)). **Campos com esses nomes no corpo são ignorados.**

## Ciclo de vida do relato

```
ABERTO ──(equipe responde)──▶ RESPONDIDO ──(usuário responde)──▶ ABERTO …
  │                               │
  └─────────(equipe fecha)────────┴──────────▶ FECHADO
```

- **ABERTO** — o relato acabou de ser criado, ou o usuário escreveu depois da última resposta da equipe.
- **RESPONDIDO** — a equipe respondeu e aguarda o usuário.
- **FECHADO** — só a equipe fecha. Relato fechado **não aceita mensagem** (`409`); o usuário abre outro.

**Não lido.** Para o usuário, o relato está não lido quando tem mensagem da equipe posterior à última vez que
ele o marcou como lido. Para a equipe vale o simétrico. Quem escreve uma mensagem também conta como tendo lido.

## O que o monitor faz com o conteúdo

- **Diagnóstico da tela** (o `formato` versiona a carga): é guardado como veio, até 64 KB. O usuário **não o
  recebe de volta**; só a equipe o vê.
- **Texto livre** (título e mensagens): o servidor troca **credenciais** (`Bearer …` e tokens de acesso
  soltos) por `[credencial removida]`. E-mail e CPF ficam, porque o usuário pode querer ser contatado por eles.
- **Instantes** são os do servidor. O `em` que a casca manda nas mensagens é ignorado.

## Retenção

A retenção conta a partir do **fechamento**, e relato aberto não expira:

- **30 dias** depois de fechado, o diagnóstico é apagado;
- **90 dias** depois de fechado, o relato e a conversa são apagados.

## Quem, na equipe, enxerga o relato

A equipe enxerga o relato pela **organização do relator**, que vem no cabeçalho `X-Relator-Org`. O valor é o
**nome da organização** (ex.: `acme`), sem sufixo. O monitor o compara com as organizações **ativas** em que o
membro da equipe tem `API_MASTER` ou `API_ENGINEER`, conforme o token de acesso dele ([claims](../auth/README.md)). Se o
nome vier noutro formato, **nenhum** membro da organização vê o relato, só o operador.

## Documentos

- [Endpoints de relato](endpoints/relatos.md) · [Erros](erros.md) · [Exemplos](exemplos.md)
