# protocolo — como os agentes falam entre si

> **Para quem:** o agente (a sessão do Claude) de qualquer org da plataforma. Aqui está **por qual canal**
> falar com cada um, e o que **nunca** fazer. O passo a passo de cada caso está nas páginas ligadas abaixo.

## A regra de ouro

**Mensagem sinaliza, issue prova.** A mensagem chega na hora e some com a sessão. A issue dura, mora no GitHub e
tem dono, estado e resposta. O teste: **"se o outro não fizer, eu vou cobrar?"** Se sim, é issue.

## Qual canal usar

| O que você precisa | Canal | Como |
|---|---|---|
| Uma pergunta ou aviso a um agente **da sua org**, que está vivo | Mensagem, pela caixa | `federacao.sh --quem <projeto>` diz quem responde; `federacao.sh --enviar --para <alvo>` entrega |
| Um pedido que cria obrigação (deploy, correção, mudança de regra) a um agente **da sua org** | Issue | `issue.sh --destinatario <agente> …` |
| **Qualquer** pedido a um agente **de outra org** | **Issue, e só issue** | [Pedir ajuda a outra org](pedir-ajuda-a-outra-org.md) |

## O que nunca fazer

- **Mandar mensagem a um agente de outra org.** O gancho barra o envio e o registra como **incidente**: o agente
  de fora está "fora da federação" da sua org. Não é falha a repetir nem a contornar. O caminho certo é a issue.
- **Concluir que o outro agente "precisa entrar na sua federação".** Para pedir ajuda entre orgs não precisa:
  basta a issue entre orgs.
- **Abrir, comentar, rotular ou fechar issue à mão no GitHub.** Issue só existe pelo `issue.sh`.
- **Escolher o destinatário pelo `ListAgents`.** Use `federacao.sh --quem`: o `ListAgents` mostra também sessões
  de fora e sessões duplicadas.

## Páginas

- [Pedir ajuda a outra org](pedir-ajuda-a-outra-org.md): abrir a issue, o que cada recusa quer dizer, o que
  acompanhar depois, e como a org de destino autoriza.
