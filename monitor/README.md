# monitor — relatos de suporte (guia)

> **Papel:** guarda os **relatos de suporte** que os usuários enviam e a **conversa** entre o usuário e a
> equipe da plataforma sobre cada relato. Quem grava e lê em nome do usuário é um **serviço chamador**, com
> **credencial de serviço** própria: o BFF da aplicação (casca + BFF) e, desde 2026-10-07, o KORC. Quem atende
> é a **equipe** (`API_MASTER`/`API_ENGINEER`), pela tela do monitor. O relato **não** vira ticket em outro
> sistema.
> Pré-requisitos: [conceitos](../02-conceitos.md), [borda](../gateway/README.md), [BFF](../bff/README.md).

## Quem chama o quê

| Quem | Por onde | Como se identifica | O que alcança |
|---|---|---|---|
| **Serviço chamador** (o BFF, o KORC), em nome do usuário | `/v3/monitor/suporte/v1/…`, pela borda | credencial de serviço própria + cabeçalhos do relator | só os relatos **daquele usuário** |
| **Equipe da plataforma** | tela do monitor | token de acesso com `API_MASTER` ou `API_ENGINEER` | os relatos **da própria organização**; o operador vê todos |

**O browser nunca fala com o monitor.** O relator não se declara: o serviço chamador tira da própria sessão,
no servidor, o usuário, a organização e o tenant, e os manda em cabeçalhos
([endpoints](endpoints/relatos.md#cabeçalhos)). **Campos com esses nomes no corpo são ignorados.**

## Mais de um chamador

Desde 2026-10-07 o monitor tem dois chamadores, o BFF e o KORC, e pode ter outros. O que isso muda:

- **Cada chamador tem a própria credencial de serviço.** O relato guarda qual delas o gravou, e a equipe vê
  esse nome no relato (campo `cliente`). O usuário não o vê.
- **A rota é alcançável de fora da rede da plataforma, pela borda**, desde 2026-10-07. O chamador não precisa
  rodar dentro da plataforma. Pela borda, ele manda também o cabeçalho de identificação que ela exige
  ([endpoints](endpoints/relatos.md#cabeçalhos)).
- **A credencial não é limitada por usuário.** Quem a tem grava e lê em nome de qualquer usuário, porque o
  monitor confia no que o chamador diz nos cabeçalhos. Por isso o chamador tira o relator da própria sessão,
  no servidor, e nunca de dado que o cliente mande; e a credencial nunca chega ao browser.
- **O alcance é por usuário, não por chamador.** "Os relatos do usuário" são todos os gravados com aquele
  `X-Relator-Username`, por qualquer chamador: um relato aberto pelo KORC aparece na lista do mesmo usuário na
  aplicação, e o inverso. A lista não diz por qual chamador cada relato entrou; o chamador que precisa separar
  os seus guarda os `id` que criou.

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

- **Diagnóstico** (o `formato` versiona a carga): é guardado como veio, até 64 KB. O usuário **não o
  recebe de volta**; só a equipe o vê. O `formato` 1 é o da tela da aplicação
  ([o que leva](../shell/seguranca.md#diagnóstico-do-relato-de-suporte)), e a tela da equipe o interpreta.
  Outro chamador usa outro inteiro em `formato`: a carga é guardada do mesmo jeito, e a equipe a vê como JSON.
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
