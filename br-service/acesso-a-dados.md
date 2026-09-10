# br-service · acesso a dados de dentro do processador

> **A pergunta que este documento responde:** *meu processador precisa ler um dado que **não é** do
> usuário que disparou o comando — o que eu uso?*
> Guia: [README.md](README.md). Em qual contexto você está: [contextos.md](contextos.md).

## Contents
- A resposta, primeiro
- Por que a escolha errada não dá erro
- Os quatro controles de acesso, e quando cada um morde
- A projeção que não aparece
- Por que retentar não conserta
- A partir de quando isto vale

---

## A resposta, primeiro

Uma projeção pode declarar **recorte de leitura por titular**: quem consulta vê apenas as linhas de que
é titular. Isso muda o que o seu processador recebe, e a escolha é sua:

| O processador precisa ver… | Use | Efeito |
|---|---|---|
| **só o que é do usuário** que disparou o comando | o `authToken` do corpo, na superfície segura | **o recorte se aplica** |
| **além do usuário** — validar contra dado de terceiro | o **endpoint interno de cluster** | não há usuário, e **não há recorte** |

**No caminho assíncrono não existe escolha:** não há JWT nenhum, e o endpoint interno é o único caminho.
Não procure por um token no payload — [não há](contextos.md), em nenhum dos dois contextos assíncronos.

O endpoint interno pede **`X-Tenant-Id`** e **nenhum `Authorization`**; o path é injetado pela plataforma
no ambiente do processador. ⛔ **Nunca fixe o path no código, nem embuta um JWT.**

A forma do corpo da consulta, os controles de paginação e ordenação e os códigos de retorno são os
mesmos dos dois caminhos, e estão em [persistence-q — consulta](../persistence-q/endpoints/consulta.md).
O que muda entre eles é **só o envelope de autorização**.

## Por que a escolha errada não dá erro

Usar o token do usuário para uma regra que precisa enxergar dado de outra pessoa devolve **lista vazia
com `200`** — não uma recusa.

E `204` continua querendo dizer "nenhum resultado". Sob recorte, "nenhum resultado" pode significar
"nenhuma linha é sua", e **não há como distinguir os dois casos pela resposta**.

Para uma invariante que testa exatamente "essa pessoa existe?", os dois casos são indistinguíveis: a
regra conclui que o dado não existe e recusa um comando legítimo. É o engano mais fácil de cometer aqui,
e ele não aparece em log nenhum.

> **Regra prática:** se a sua invariante compara com dado de **terceiro** — outro titular, outro
> responsável, outra ficha —, o caminho é o **endpoint interno**, mesmo que você tenha um token em mãos.

O efeito do recorte sobre a consulta em si — acúmulo de papéis, associações, `_count` — é fatia do
persistence-q, na seção *"Quando quem consulta é um processor, e não uma pessoa"* de
[persistence-q](../persistence-q/README.md). Aqui se descreve só o que muda **na escrita do processador**.

## Os quatro controles de acesso, e quando cada um morde

O processador não é o único ponto onde o acesso é avaliado. Saber **em que instante** cada controle roda
é o que separa "meu processador está errado" de "o modelo está incompleto".

| Controle | Quando roda | O que o autor do processador vê |
|---|---|---|
| **papéis do comando** | **antes** de o processador ser chamado | nada. **Se o seu processador rodou, esta autorização já passou** — não a reimplemente |
| **recorte de leitura** (`accessControl.read` + `scope.read`) | na consulta que **o processador** faz | é a escolha da seção anterior |
| **controle de escrita** (`accessControl.write`) | na **materialização da projeção**, já depois de o processador ter retornado | o comando responde `200`, o processador roda inteiro, **e a projeção não aparece** |
| **leitura da entidade associada** | na criação, quando o dado traz associação | recusa numa entidade que você não estava consultando de propósito |

Os dois primeiros são conhecidos; os dois últimos são os que produzem sintoma sem culpado aparente.
Quem declara qualquer um deles é o forger, no `_conf` da entidade — ver
[forger — entity](../forger/endpoints/entity.md).

## A projeção que não aparece

O sintoma é este, e ele **não é falha do seu processador**:

> o comando responde `200` · o processador executa e devolve normalmente · a projeção nunca reflete a
> mudança · nada no caminho do comando indica erro.

Quando o autor do evento não tem papel no `accessControl.write` da entidade, a materialização é recusada
**depois** de todo o resto ter dado certo. O caminho do comando já respondeu, então a recusa não tem por
onde voltar ao cliente.

**Como confirmar, sem entrar em contêiner algum:** o desfecho do consumo aparece na consulta de log do
persistence-crs, que é endpoint de cliente e é autorizada pelo **segredo do tenant** — não pelo token de
domínio. Procure pelo termo do erro de consumo no intervalo em que o comando rodou; o registro nomeia o
autor recusado, a projeção e o evento. Requisição, cabeçalhos e formatos de intervalo em
[persistence-crs — consulta de logs](../persistence-crs/endpoints/logs.md).

**A correção é no modelo, não no processador:** o papel do autor precisa constar do `accessControl.write`
da entidade de destino. **Audite essa lista antes** de declarar controle de escrita em qualquer entidade
que já receba projeção — é mais barato que diagnosticar depois.

## Por que retentar não conserta

Recusa por papel **não é falha transitória**. A mensagem volta para a fila e falha de novo, igual, em
toda tentativa, até esgotar as tentativas e ir para a fila de descarte. Nenhuma delas passa a ter o papel
que falta.

A consequência prática: **repetir o comando não resolve, e piora o diagnóstico** — cada repetição produz
mais um evento que vai falhar do mesmo jeito. Corrija a lista de papéis primeiro.

## A partir de quando isto vale

Tudo o que este documento descreve — o recorte de leitura, o endpoint interno **não** recortar, e o
controle de escrita na materialização — vale a partir de **`amd64-260909b`**.

⚠️ **A versão importa aqui mais do que em qualquer outra página desta fatia.** Houve **duas** imagens no
mesmo dia. Na primeira, uma entidade com recorte alcançada pelo endpoint interno era **recusada com
`403`** — declarar o recorte quebraria todo processador que lesse por ali. Foi na segunda que a recusa
saiu e o endpoint interno passou a simplesmente não recortar.

> **Ressalva honesta, e ela não é formalidade:** o controle de escrita na materialização está **no
> artefato** e **ainda não foi exercitado** por um comando autenticado que materialize projeção.
> "Presente no artefato" e "comprovado em execução" não são a mesma coisa. Trate o sintoma descrito acima
> como o comportamento esperado, e não como comportamento observado.
