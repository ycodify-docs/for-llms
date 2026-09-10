# br-service · os três contextos de invocação

> **O fato que decide tudo o mais:** o br-service é chamado de **três** lugares diferentes, e os três
> mandam um corpo diferente, entregam um número diferente de argumentos ao processador e esperam uma
> **forma de resposta diferente**. Escolher o contrato errado não dá erro — dá `200` e nada acontece.
> Guia: [README.md](README.md). Autoria do arquivo: [processadores.md](processadores.md).
> Leitura de dados de dentro do processador: [acesso-a-dados.md](acesso-a-dados.md).

## Contents
- Qual é o seu caso
- 1 · Regra de negócio (síncrono)
- 2 · Coordenação / saga (assíncrono)
- 3 · Projeção cross-contexto (assíncrono)
- Quando o retorno some sem erro
- Reentrega e idempotência

---

## Qual é o seu caso

**Quem decide o contexto é o modelo, não o nome do arquivo.** O sufixo da rota
(`…/coordination/<n>`, `…/projection/<n>`) é convenção de leitura; o que realmente determina como o seu
processador é chamado é **onde a rota foi declarada** no `.model.json`.

| | **1 · Regra de negócio** | **2 · Coordenação (saga)** | **3 · Projeção cross-contexto** |
|---|---|---|---|
| Declarado em | `command.<cmd>.br.route` | `event.<ev>.domainBus.triggerCoordination[].br.route` | `event.<ev>.domainBus.triggerProjection[].br.route` |
| Quando roda | **durante** o comando, antes de o evento ser gravado | **depois** do evento, por fila | **depois** do evento, por fila |
| Síncrono? | sim | não | não |
| Argumentos recebidos | **três** | **um** | **um** |
| Há JWT do usuário? | **sim**, em `authToken` | **não** — nunca | **não** — nunca |
| Resposta esperada | objeto **direto**, sem envelope | `{"processedData": {"targetCommand": {…}}}` | `{"processedData": {"<projeção>": {…}}}` |
| Se o processador falhar | **o comando é abortado** | mensagem devolvida à fila | mensagem devolvida à fila |
| Tempo máximo | **17 s** | **30 s** | **17 s** |

> ⚠️ **O envelope `processedData` é obrigatório nos dois contextos assíncronos e proibido no síncrono.**
> É o engano mais caro desta fatia: um processador de coordenação que devolve o objeto direto recebe
> `200`, e o comando-alvo simplesmente não nasce — sem erro, sem registro visível ao autor.

---

## 1 · Regra de negócio (síncrono)

Roda **dentro** do comando, depois da autorização por papel e **antes** de o evento existir. É o único
contexto em que o processador pode **abortar** a operação.

### O que chega

```json
{
  "route": "<rota>",
  "data": { "<atributos do comando>": "..." },
  "authToken": "<token de acesso do usuário, ou null>",
  "tenantIds": ["<tenant do modelo de leitura>"]
}
```

> Ilustrativo — `<...>` é placeholder, não JSON literal.

- **`data` é só o payload do comando** — não o estado do agregado. E já vem **filtrado**: atributo que
  não está declarado no comando é removido antes de chegar aqui.
- **`authToken` é o cabeçalho de autorização inteiro**, como o cliente o enviou.
- **`tenantIds` tem exatamente um elemento**, e ele vem do **modelo** (o tenant do modelo de leitura do
  agregado) — **não** é o tenant do cabeçalho da requisição. Os dois normalmente coincidem, mas são
  origens diferentes e podem divergir.
- O processador é chamado com **três argumentos**: `(data, authToken, tenantIds)`.
  ⚠️ Se `authToken` vier nulo — e vem, em toda operação que não passa por um usuário autenticado — o
  processador é chamado com **um argumento só**. Quem lê o segundo parâmetro sem checar recebe
  `undefined`.

### O que devolver

O objeto do próprio comando, **sem envelope**. Ele é mesclado de volta antes de o evento ser gravado.

**A mescla é por whitelist, e ela descarta em silêncio:**

| O que você devolve | O que acontece |
|---|---|
| chave **que já existia** no comando, com valor | substitui o valor — é o enriquecimento |
| chave **que já existia**, com valor nulo | **ignorada**; o valor original sobrevive |
| chave **nova**, que não existia no comando | **descartada**; não chega ao evento |

Por isso **todo campo que o processador pretende calcular já precisa existir como atributo do comando no
modelo**. Devolver um campo novo não é erro: é um valor que se perde sem aviso.

### Se falhar

Lançar aborta o comando: nenhum evento é gravado e nada segue para as filas.

⚠️ **O código que o cliente recebe não é o que o br-service respondeu.** O processador que lança faz o
br-service responder `400`; o motor de comandos, porém, embrulha essa falha e o cliente final recebe
**`510`**. A mensagem do processador viaja dentro do corpo dessa resposta. Quem trata `400` no cliente
nunca a encontra. *(A recusa por papel é a exceção: essa chega como `403`.)*

---

## 2 · Coordenação / saga (assíncrono)

Roda depois de o evento existir, a partir da fila de coordenação. Serve para que um evento num agregado
faça nascer um comando em outro.

### O que chega

```json
{
  "route": "<rota>",
  "data": {
    "<entidade de origem>": { "aggregateid": "<uuid>", "...": "..." },
    "aggregateState": { "...": "..." },
    "targetTenantId": "<uuid>",
    "sourceTenantId": "<uuid>"
  }
}
```

- **Um argumento só.** Não há `authToken` e não há `tenantIds`.
- **A entidade de origem é a primeira chave** de `data`, e o objeto dentro dela é o payload do evento,
  já com `aggregateid` incluído.
- **`aggregateState` é o estado do agregado de origem, reconstituído.** ⚠️ Ele pode chegar **`{}`, sem
  nenhum aviso**, quando o agregado de origem não é conhecido no tenant de destino — o que é comum
  justamente no caso cross-contexto. **Trate `{}` como normal** e caia para o payload do evento.
- Os dois tenants vêm **no corpo**, não em cabeçalho: nenhum cabeçalho de identificação chega aqui.

### O que devolver

```json
{ "processedData": { "targetCommand": {
    "boundedContext": "<bc destino>",
    "aggregateType":  "<agregado destino>",
    "commandName":    "<comando>",
    "data":           { "...": "..." }
} } }
```

Os quatro campos são **obrigatórios**; faltar um ou trocar o tipo derruba o consumo da mensagem.

> ⚠️ **Um comando, nunca uma lista.** `targetCommand` é lido como objeto único — devolver um array não
> dispara N comandos, derruba o consumo. **Não existe fan-out**: uma coordenação emite exatamente um
> comando. Precisar encerrar N filhos de um cancelamento não tem caminho por aqui hoje.

O comando resultante é submetido **sem credencial** — ele não carrega o usuário que originou a cadeia.

---

## 3 · Projeção cross-contexto (assíncrono)

Roda depois do evento, a partir da fila de projeção cross-contexto. Serve para traduzir o evento de um
contexto na linha da projeção de outro.

### O que chega

```json
{
  "route": "<rota>",
  "data": {
    "<entidade de origem>": { "aggregateid": "<uuid>", "...": "..." },
    "_meta": {
      "targetTenantId": "<uuid>",
      "sourceTenantId": "<uuid>",
      "targetSchema":   "<esquema destino, quando declarado>"
    }
  }
}
```

- **Um argumento só**, e o contexto viaja em **`_meta`** — repare que aqui os tenants estão aninhados,
  ao contrário do contexto de coordenação, onde ficam no nível de cima.
- Não há `authToken`. Nenhum cabeçalho de identificação chega aqui.

### O que devolver

```json
{ "processedData": { "<nome da projeção destino>": {
    "aggregateid": "<uuid>",
    "...": "..."
} } }
```

- **A chave é o nome da projeção de destino** — é por ela que a linha é procurada e gravada. Devolver a
  entidade de origem grava na projeção errada.
- **`aggregateid` é obrigatório.** É a chave que decide entre **criar** e **atualizar**: existindo linha
  com aquele `aggregateid`, o retorno é aplicado sobre ela; não existindo, uma linha nova é criada. Sem
  o campo, o consumo da mensagem falha.
- **Uma projeção por retorno.** Só a primeira chave é considerada.

---

## Quando o retorno some sem erro

Três formas de o processador rodar, responder `200`, e nada acontecer. Nenhuma delas produz erro para o
autor, e é por isso que estão juntas aqui.

| Contexto | O que faz sumir | Como se manifesta |
|---|---|---|
| regra de negócio | devolver **chave que não existe** no comando | o evento é gravado sem o campo calculado |
| coordenação | devolver **sem o envelope**, ou sem `targetCommand` | a mensagem é confirmada e a saga é **descartada**; o comando-alvo nunca nasce |
| projeção | devolver **sem o envelope** | o payload do evento segue **sem transformação** para a projeção, e normalmente falha depois por coluna desconhecida |

**Como confirmar de fora, sem entrar em nenhum contêiner:** o desfecho do consumo — inclusive a recusa
por papel na materialização — aparece na consulta de log do persistence-crs, autorizada pelo segredo do
tenant, não pelo token de usuário. Ver
[persistence-crs — consulta de logs](../persistence-crs/endpoints/logs.md) e, para o caso específico da
projeção que não aparece, [acesso-a-dados.md](acesso-a-dados.md).

---

## Reentrega e idempotência

- **Entrega ao-menos-uma-vez nos dois contextos assíncronos.** A mesma rota, com os mesmos dados, pode
  chegar mais de uma vez. Processador de coordenação e de projeção **precisa** ser idempotente — a chave
  natural é o `aggregateid` do payload do evento.
- **Falha é retentada um número limitado de vezes** e, esgotadas as tentativas, a mensagem vai para a
  fila de descarte. Retentar só ajuda em falha transitória: recusa por papel ou modelo incompleto volta
  a falhar igual em toda tentativa.
- **O contexto síncrono não é retentado** — falhar ali aborta o comando, e quem repete é o cliente.
- **Estoure o tempo e a chamada é falha**, com a mesma consequência de uma exceção. Os tetos estão na
  tabela do topo; leitura de enriquecimento cabe neles com folga, integração externa lenta não.
