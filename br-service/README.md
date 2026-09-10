# br-service — guia do serviço

> **Papel:** executa as **funções** que o modelo declara — regra de negócio de um comando, coordenação
> entre agregados (saga) e transformação para projeção de outro contexto. Recebe dados e devolve dados.
> **Quem chama:** **somente** o [persistence-crs](../persistence-crs/README.md) — nunca o cliente final
> diretamente (CP-6). Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Qual é a sua pergunta

| A pergunta | Onde ela é respondida |
|---|---|
| em que contexto o meu processador roda, e **o que chega** nele | [contextos.md](contextos.md) |
| **o que eu devolvo** para o retorno não ser descartado | [contextos.md](contextos.md) |
| o comando respondeu `200` e **nada aconteceu** | [contextos.md — quando o retorno some sem erro](contextos.md) |
| meu processador precisa **ler a ficha de outra pessoa** | [acesso-a-dados.md](acesso-a-dados.md) |
| a **projeção não aparece** depois de um comando que deu certo | [acesso-a-dados.md](acesso-a-dados.md) |
| **publiquei e a rota não existe** | [processadores.md](processadores.md) |
| como se escreve e se publica o arquivo | [processadores.md](processadores.md) |

## Contents
- Posição no fluxo
- Índice de endpoints
- Forma canônica da rota
- Contrato de resposta
- Ciclo de vida de uma requisição
- Callback do processor
- Pontos de coordenação
- Pitfalls

---

## Posição no fluxo

Quando o **model** de um comando ou de um evento declara uma **rota** de br, o persistence-crs chama o
br-service nessa rota, passando os dados; o serviço executa a **função** correspondente e devolve o
resultado, que o persistence-crs incorpora ao processamento (CP-6/CP-7). O br-service **não é chamado
diretamente pelo cliente**.

**São três contextos de invocação, e eles não têm o mesmo contrato** — corpo, número de argumentos,
forma da resposta e tratamento de falha mudam entre eles. É o que
[contextos.md](contextos.md) descreve, e é a leitura obrigatória antes de escrever qualquer processador.

Um processador **pode ler de volta** persistence-q/crs para enriquecimento — como, e com que identidade,
está em [acesso-a-dados.md](acesso-a-dados.md). Efeito externo com escrita é aceitável nos hooks
assíncronos, desde que tolere reentrega; no **caminho crítico do comando**, evite.

## Índice de endpoints

> Endpoints de saúde/observabilidade **não** constam aqui.

| Operação | Método · Path | Documento |
|---|---|---|
| Executar função (rota) | `POST /br` | [endpoints/br.md](endpoints/br.md) |
| Implantar processadores de uma organização | `POST /processors/deploy/{org}` | [endpoints/processors-deploy.md](endpoints/processors-deploy.md) |
| Desfazer a última publicação | `POST /processors/rollback/{org}/{backupId}` | [endpoints/processors-deploy.md](endpoints/processors-deploy.md) |
| Consultar o log de execução | `GET /logs/query/…` | [endpoints/logs.md](endpoints/logs.md) |

> **As rotas administrativas exigem um cabeçalho a mais.** Publicação, rollback e consulta de log passam
> pela borda como rotas administrativas e pedem `X-Forger-Credential` — que **nunca** vai junto com
> `X-Tenant-Id`. O `POST /br` não é afetado: ele é chamado pelo motor de comandos dentro do cluster.

> **⚠️ `POST /coordination` — pendente (gap de plataforma).** O motor de comandos aciona uma coordenação
> **síncrona no caminho do comando** quando o modelo do comando declara `command.<c>.coordination.route`,
> postando em **`POST /coordination`** (corpo `{content:{route,data}}`). **O br-service NÃO implementa
> este endpoint** — só `/br`. Declarar `coordination.route` hoje não é um no-op silencioso: a chamada
> falha e leva o comando junto. Para coordenação, use `event.domainBus.triggerCoordination`
> ([contextos.md](contextos.md)).

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md).

## Forma canônica da rota (obrigatória)

O br-service serve **múltiplas organizações**, cada uma com uma hierarquia de recursos semelhante. Para
**evitar colisão** entre funções de organizações diferentes, a rota é **totalmente qualificada**, do
mais geral ao mais específico:

```
<org>/<project>/<boundedContext>/<aggregate>/<função>
```

- **Regra de negócio:** `<org>/<project>/<bc>/<aggregate>/<comando>`
- **Coordenação:** `<org>/<project>/<bc>/<aggregate>/coordination/<nome>`
- **Projeção:** `<org>/<project>/<bc>/<aggregate>/projection/<nome>`

**Por que isso garante unicidade global** — cada segmento é único **dentro do seu pai**:

| Segmento | Único dentro de |
|---|---|
| `org` | espaço de nomes global |
| `project` | da organização |
| `boundedContext` | do projeto |
| `aggregate` | do bounded context |
| `função` (comando / coordination/nome / projection/nome) | do agregado |

Aninhando os escopos, o **path completo é globalmente único** → duas organizações podem ter o mesmo
`pedido/criar` sem conflito, pois os prefixos `<org>/<project>/<bc>` diferem.

> Pela [convenção padrão](../02-conceitos.md#convenção-padrão-de-topologia-project--bounded-context--esquema)
> (1 project = 1 bounded context), os segmentos `project` e `boundedContext` costumam **coincidir** —
> ex.: `acme/vendas/vendas/pedido/criar`. Não é erro: são escopos distintos que, por padrão, têm o mesmo nome.

> ⚠️ **O sufixo é convenção de leitura, não é o que decide o comportamento.** Quem determina em que
> contexto o processador roda é **onde a rota foi declarada no modelo** — ver
> [contextos.md](contextos.md). Uma rota com sufixo `coordination` declarada em `command.br.route` roda
> como regra de negócio.

## Contrato de resposta

- Sucesso (`200`): o serviço devolve **diretamente** o que o processador retornou, sem envelope próprio.
- Falha (`400`): objeto com `status`, mensagem e tipo do erro.

⚠️ **O que o processador deve retornar depende do contexto**, e não é o mesmo nos três: o contexto
síncrono espera o objeto do comando **sem envelope**; os dois assíncronos exigem o envelope
**`processedData`**. A forma exata de cada um está em [contextos.md](contextos.md) — errar isso devolve
`200` e descarta o resultado em silêncio.

## Ciclo de vida de uma requisição

`POST /br`:

1. **Validação** — corpo não-nulo, objeto, com `route`. Falha → `400`.
2. **Roteamento** — localiza o processador pela `route`. Não encontrado → `400` (lista rotas disponíveis).
3. **Execução** — invoca o processador com os argumentos que o corpo determina
   ([contextos.md](contextos.md)). Pode ser assíncrono; o resultado é aguardado.
4. **Resposta** — `200` com o resultado **direto** do processador; exceção no processador → `400` com
   `{ status, mensagem, tipo }`.

### Callback do processor → persistence-q / persistence-crs (endpoint interno de cluster)

Movido, e ampliado, para **[acesso-a-dados.md](acesso-a-dados.md)** — que responde também *com que
identidade* a leitura acontece, por que o token do usuário pode devolver lista vazia com `200`, e o que
fazer quando a projeção não aparece.

O essencial, para quem só passa por aqui: hook **assíncrono** usa o **endpoint interno de cluster**
(`X-Tenant-Id`, **sem `Authorization`**); hook **síncrono** pode usar a superfície segura com o
`authToken` do corpo, **e nesse caso o recorte de leitura por titular se aplica**. ⛔ Nunca hardcodar um
JWT nem o path do endpoint interno — ambos vêm do ambiente injetado no deploy.

## Pontos de coordenação
- **CP-6** — o persistence-crs chama o br-service quando o modelo do comando exige regra/coordenação.
- **CP-7** — no fluxo de saga, a coordenação devolvida pode levar o persistence-crs a submeter um novo comando.
Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] **Saber em que contexto o processador roda** antes de escrevê-lo — é o que define argumentos e
      retorno ([contextos.md](contextos.md)).
- [ ] **Envelope `processedData` nos contextos assíncronos**, e **nunca** no síncrono.
- [ ] Na regra de negócio, **todo campo calculado já precisa existir** como atributo do comando: chave
      nova é descartada em silêncio.
- [ ] Na projeção, devolver a chave da **projeção de destino**, com `aggregateid`.
- [ ] Ler dado de terceiro → **endpoint interno**, nunca o token do usuário
      ([acesso-a-dados.md](acesso-a-dados.md)).
- [ ] Processador assíncrono **idempotente** — a entrega é ao-menos-uma-vez.
- [ ] Não chamar o br-service diretamente: ele é acionado **pelo persistence-crs** conforme o model.
- [ ] Garantir que a **rota** referenciada no model exista como processador publicado.
