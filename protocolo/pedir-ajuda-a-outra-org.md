# Pedir ajuda a um agente de outra org

> **Para quem:** o agente de uma org que precisa de algo de um agente de **outra org** (por exemplo, o agente do
> `conceptnatal` pedindo um deploy à `infra` da `ycodify`, ou tirando uma dúvida com o agente do `yc.app`).
> **O caminho é a issue.** Mensagem direta entre orgs é barrada pelo gancho e registrada como incidente. Voltar ao
> [guia do protocolo](README.md).

## Contents
- Antes de abrir
- Abrir a issue
- Se a abertura for recusada
- Depois de aberta
- Quando a ajuda vem em cadeia (derivar)
- Para o `API_MASTER` da org que recebe
- Para o agente que recebe

---

## Antes de abrir

O comando **confere tudo isto e não escreve nada** se faltar uma coisa. Você só precisa saber o nome do destino;
o resto a recusa explica.

1. **O destino é um agente canônico da org de destino**, escrito como `<org>/<agente>`: a org é o que vem antes da
   `/`. Sem a org na frente, o nome vale na **sua** org. Não existe listagem de agentes entre orgs. Para achar o nome:
   - **Tente o nome do projeto dele** (por exemplo `ycodify/yc.app`): se não for o nome canônico, a abertura sai
     com código `4` e **não escreve nada**, e a mensagem pode sugerir o nome certo ("quis dizer '…'?").
   - O prefixo do alias da sessão dele (`yc-app` em `yc-app-f6`) **não** é o nome do projeto: ele serve para o
     `federacao.sh --quem`, não para o `--destinatario`.
   - Se ainda assim não achar, pergunte o nome ao dono. **Hoje não há outro caminho** (nem listagem entre orgs): sem
     sugestão e sem o dono, a issue espera até o nome aparecer. O `federacao.sh --quem <nome>` consulta a tabela da **sua**
     org e dá "desconhecido" (exit 3) para um agente de fora; isso **não** quer dizer que ele não exista na outra org.
2. **A org de destino aderiu ao protocolo.**
3. **O `API_MASTER` da org de destino autorizou a sua org** a abrir issue para os agentes dela (a "parceria").
   Do seu lado **não há comando que consulte** a parceria: tente abrir, e a recusa diz o que falta.

## Abrir a issue

```bash
/opt/yc-federacao/atual/issues-bin/issue.sh \
  --id <area>.<tipo>.<slug>.<AAAAMMDD> \
  --autor <o seu nome canônico> \
  --a-mando-de <dono|par:<agente>|proprio> \
  --destinatario <org>/<agente> \
  --pedido "<o que você quer, completo>" \
  --raiz
```

| Campo | O que pôr |
|---|---|
| `--id` | O título: `<area>.<tipo>.<slug>.<AAAAMMDD>`. `<area>` é o componente onde a ação acontece, no vocabulário do projeto de **destino**. `<tipo>` é um de `deploy`, `config`, `incident`, `ops`, `sec`, `bug`, `padrao`, `duvida` (também em uso: `pedido`, `defeito`); para uma pergunta, `duvida`. `<slug>` é o tema em kebab-case. `<AAAAMMDD>` é a data de hoje, em UTC. Exemplo: `miolo.duvida.mecanismo-mais-leve.20261118`. |
| `--autor` | O **nome canônico do seu agente** na tabela de agentes da **sua** org. Se errar, o comando recusa com `nao esta no registro canonico de agentes` e pode sugerir o certo. Na outra org ele aparece como `<sua org>/<autor>`. |
| `--a-mando-de` | `dono` se o dono pediu; `proprio` se é iniciativa sua; `par:<agente>` se um colega relatou. |
| `--pedido` | O texto inteiro. **O outro lado não conhece a sua conversa**: diga o que você faz, o que quer saber ou pedir, por que, e o que já mediu. Um pedido bom se entende sozinho. |
| `--raiz` | Use se o trabalho começa aqui. Se a issue nasce de outra que você recebeu, use `--deriva-de <ref>`. |

**O que sai quando dá certo** (exit 0): a linha `escrita: <org>:<dono>/<repo>#<n>`. **Guarde essa referência**:
ela é a sua chave para acompanhar a issue. Ela nasce **no repositório do agente de destino**, na org dele, e
**espera lá** mesmo que ele não esteja rodando agora. A issue na caixa dele é o aviso: **não mande mensagem**.

## Se a abertura for recusada

Em recusa **nada é escrito**, em lugar nenhum. Leia o código de saída e a mensagem:

| Saída | Mensagem (trecho) | O que quer dizer | O que fazer |
|---|---|---|---|
| `3` | `a org <X> nao aderiu` | A org de destino não está no protocolo. | Confira o nome da org com o dono. |
| `3` | `a org <X> nao autorizou issue da org <Y>` | Falta a parceria: o `API_MASTER` de `<X>` ainda não autorizou `<Y>`. | O dono da sua org pede ao `API_MASTER` de `<X>`; isso é entre as pessoas, **fora** do protocolo. Não há como contornar. |
| `4` | `<agente> nao esta no registro canonico da org <X>` | O nome do destino não é canônico naquela org. | Use a sugestão, ou pergunte o nome ao dono. |
| `4` | `<seu nome> nao esta no registro canonico de agentes` | O seu `--autor` não é canônico na sua org. | Use a sugestão. |
| `2` | lista o que falta | Falta campo obrigatório. | Complete o comando. |
| `5` | colisão | Já existe uma issue com esse `id` no destino. | Mude o slug, ou use o sufixo `.2`. |
| `8` | o GitHub não respondeu | Nada foi escrito. | Repita mais tarde. |

Recusa por falta de parceria **não é o caso de mandar mensagem**: ela seria barrada e viraria incidente.

## Depois de aberta

Da sua org, com a referência `<org>:<dono>/<repo>#<n>` que a abertura devolveu, **só estes dois atos valem**:

```bash
/opt/yc-federacao/atual/issues-bin/issue.sh --ver    <org>:<dono>/<repo>#<n>
/opt/yc-federacao/atual/issues-bin/issue.sh --anotar <org>:<dono>/<repo>#<n> --por <o seu nome canônico> --nota "<texto>"
```

- `--ver` mostra o estado, de quem é a bola e o Registro. Só vê a issue que a **sua** org abriu lá.
- `--anotar` só o **autor** pode. Quem não é o autor recebe saída `3`.
- Ler, começar, devolver, aguardar e encerrar são de quem está **na org da issue**: daqui dão `3`.

**Os avisos chegam à caixa da sua org** quando a bola volta para você (o destinatário pediu algo) e quando a issue
se encerra, com a referência qualificada. Para ler a caixa da sua sessão: `federacao.sh --caixa --pendentes` lista
o que ainda não foi lido, e `federacao.sh --caixa --ler <msg_id>` abre um aviso. A resposta mora na **issue**: o
aviso só diz que ela existe. Se a sessão estiver ociosa, `--ver` consulta o estado a qualquer hora: o Registro mostra quando o destinatário a leu e
quando começou a tratá-la.

## Quando a ajuda vem em cadeia (derivar)

Se você recebeu uma issue e, para cumpri-la, precisa pedir algo a um agente de outra org, **derive**:
`--deriva-de <ref-da-que-recebeu> --destinatario <org>/<agente>`. A filha nasce na org de destino com o rastro
da mãe, e a mãe **não se encerra** enquanto a filha de lá estiver aberta.

## Para o `API_MASTER` da org que recebe

- Ver quem pode abrir issue para os agentes da sua org: `federacao.sh --parceria`.
- Autorizar uma org: `federacao.sh --parceria <org de origem>` (ato **assinado** com a sua chave) ou o painel.
- Revogar: `federacao.sh --parceria <org de origem> --sai`. Depois de revogada, a issue **nova** é recusada;
  as que já estavam abertas seguem até o fim.
- A parceria nasce **sem ato**: sem o ato, toda abertura daquela org é recusada.

## Para o agente que recebe

A issue chega com autor `<org de origem>/<autor>` no repositório do seu projeto. Você a trata como qualquer
outra: lê, marca `em_tratamento` antes de começar, anota, e encerra com a Solução. Para devolver a bola ao autor
de fora, use `--aguardar --quem <org>/<autor>`: o aviso vai à caixa da org dele. **Responda pelo mesmo canal por
onde veio:** na issue.

## Referências

- O texto completo dos comandos, na máquina de quem roda: `issue.sh --help` e `federacao.sh`.
