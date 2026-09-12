# persistence-q — guia do serviço

> **Papel:** **lado de leitura** do CQRS. Consulta o estado dos agregados pela ótica de suas
> **projeções** materializadas. Não executa comandos nem reprocessa eventos.
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md). Lê as projeções
> criadas pelo forger (CP-2) e mantidas pelo fluxo es-n → projeção (CP-5).

## Contents
- Pré-requisitos do chamador
- Índice de endpoints
- Ciclo de vida de uma consulta
- Depois de um comando, espere antes de consultar
- Linguagem de consulta (resumo)
- Recorte de leitura por titular (como se faz · o que torna uma linha sua · `read` × `scope` · processors)
- Pontos de coordenação
- Pitfalls

---

## Pré-requisitos do chamador

- **Cabeçalhos obrigatórios:** `Authorization` e o **cabeçalho de tenant** `X-Tenant-Id`.
- **Autorização de tenant:** o `tenant-id` deve pertencer ao usuário; senão → `403`.
- **Vocabulário do tenant:** os identificadores de consulta e predicados válidos são definidos pelo
  **model**/projeções provisionados para o tenant (não inventar predicados).
- **Dataschema em `RUNNING`, verificado uma vez por instância:** a consulta só é atendida se o
  dataschema do tenant estiver em `RUNNING` — mas essa checagem acontece **na primeira requisição
  daquele tenant em cada instância do serviço**, não a cada consulta. Depois disso a instância atende o
  tenant sem reavaliar o estado. Consequência: **voltar o dataschema para `MODELING` não interrompe**
  quem já está sendo atendido, e a mesma mudança pode "pegar" numa instância e não noutra. Se a
  alteração precisa valer imediatamente em todo lugar, trate-a como operação de implantação. Ver
  [forger/dataschema — gate de status](../forger/endpoints/dataschema.md#atualizar) e
  [query-controls §Quando o modelo é lido](query-controls.md#quando-o-modelo-é-lido).
- **Como a spec do tenant é resolvida:** igual ao write-side — dataschema `RUNNING` via Forger →
  **serviço de cache** (`../cache`) → **se falta no cache, a consulta falha** (`510`, "republique o
  modelo"). **Não há recuperação automática a partir do Forger:** o modelo entra no cache **só**
  quando é publicado, e a entrada **não expira** sozinha. Um `510` aqui significa que o modelo nunca
  foi publicado, que a entrada foi removida, ou que **o dataschema está em edição** (`MODELING`).
  **Quem publica e quem remove o modelo é o forger** — procedimento e pré-requisitos na fatia dele:
  [forger/dataschema](../forger/endpoints/dataschema.md#o-que-a-transição-faz-com-o-modelo-no-cache).
  **Não invalide cache à mão para propagar alteração:** fechar a edição do dataschema republica o
  modelo de leitura, e republicar o `.model.json` sobrescreve o de escrita. **Restart nunca foi
  remédio.** *(Este item dizia, até 2026-09-10, que era preciso "invalidar o cache — as duas chaves do
  modelo" antes de republicar. Era trabalho desnecessário.)*
  Detalhe: [persistence-crs/README §Depois de alterar o modelo](../persistence-crs/README.md).
- **Linha que não aparece não é, necessariamente, consulta errada.** Se o comando respondeu `200` e a
  linha não está aqui, a projeção pode ter falhado ao ser aplicada — o `200` responde pela escrita, não
  pela leitura. A falha fica registrada, e a receita para achá-la está em
  [persistence-crs/erros.md § O comando respondeu 200 e a linha não apareceu](../persistence-crs/erros.md#o-comando-respondeu-200-e-a-linha-não-apareceu).
- **Dois identificadores na projeção:** cada linha tem `id` (**Long**, PK da linha) e `aggregateid`
  (**UUID** de 36 chars, o id do agregado; `projecao.aggregateid == aggregate.id`). Filtra-se por
  `aggregateid` para achar a projeção de **um** agregado específico. Para operar o agregado depois
  (comando/`/history` na [persistence-crs](../persistence-crs/README.md)), usa-se o **UUID**
  (`aggregateid`), **nunca** a PK `id` (Long) — o Long num endpoint de agregado → `510`
  "Invalid UUID string".

## Índice de endpoints

> Apenas o endpoint autenticado é documentado. Endpoints internos sem autenticação e endpoints de
> observabilidade/administração **não** constam aqui.

| Operação | Método · Path | Documento |
|---|---|---|
| Consultar projeções | `POST /` | [endpoints/consulta.md](endpoints/consulta.md) |
| Controles da DSL (operadores, paging, sorting, cache, associações) | — | [query-controls.md](query-controls.md) |

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md).

## Ciclo de vida de uma consulta

`POST /`:

1. **Auth + tenant** — valida `Authorization` e o `tenant-id`.
2. **Detecção de modo** — pelo primeiro caractere do corpo: `[` → **array** (múltiplos critérios);
   `{` → **object** (critério único).
3. **Leitura das projeções** — para cada critério, lê a projeção correspondente no banco de leitura,
   aplicando os predicados informados.
4. **Resposta** — sempre um **array no topo**, na **mesma ordem** dos critérios; `200` se houver algum
   resultado, `204` se todos vierem vazios.

A consulta é **idempotente** e **não** modifica estado.

> **Consistência eventual:** a projeção é atualizada de forma assíncrona após cada comando. Logo após
> um comando bem-sucedido, a consulta pode ainda **não** refletir a mudança por um curto intervalo.

### Depois de um comando, espere antes de consultar

**Regra:** todo polling que consulta este serviço logo após um comando deve começar por um **atraso
inicial (em milissegundos)** e depois **reconsultar**, com número de tentativas limitado. **Não** dispare
a primeira consulta junto com a resposta do comando.

**Por que a regra existe:** o mecanismo que persiste a **projeção** é **desvinculado** do mecanismo que
persiste o **agregado**. São dois caminhos independentes — o comando grava o evento e responde; a
projeção é aplicada **depois**, por outro caminho (o despacho assíncrono do evento). Logo:

| O que o `200` do comando afirma | O que ele **não** afirma |
|---|---|
| o evento foi gravado e a transição era válida | que a linha da projeção já existe ou já está atualizada |

Consequências práticas:

- **Consultar imediatamente devolve o estado anterior** — e isso **não** é erro, nem do comando nem da
  consulta. Tratar esse resultado como verdade leva à conclusão errada de que o comando falhou.
- **Não há número universal para o atraso.** Ele varia com a carga do despacho e com o tamanho do
  modelo. Por isso a regra é *atraso inicial + reconsulta limitada*, nunca uma consulta única.
- **Reconsultar é seguro:** a consulta é idempotente e não modifica estado.
- **Uma falha na projeção não aparece na resposta do comando.** Se a linha nunca chegar, a causa está no
  caminho assíncrono, não na resposta que o cliente recebeu — investigar só a resposta do comando não
  encontra nada.

## Linguagem de consulta (resumo)

- Cada **critério** é um objeto com **um** identificador de consulta (rótulo escolhido pelo cliente)
  mapeando para um conjunto de **predicados** (pares chave-valor, combinados por "E").
- O rótulo aparece **literalmente na resposta**, agrupando os registros encontrados.
- O **vocabulário** de predicados é definido pelo modelo do tenant; nome desconhecido → erro de
  tradução de critério.

Detalhe e exemplos: [endpoints/consulta.md](endpoints/consulta.md).

## Recorte de leitura por titular

Uma entity pode declarar que um **papel** lê apenas as linhas de que o usuário é **titular** — o resto
da tabela deixa de existir para ele. O serviço acrescenta esse recorte ao critério que você enviou:
não é preciso pedi-lo, e não é possível desligá-lo.

> **⚠️ Versão importa, e o sufixo não é detalhe de nomenclatura.** Esta seção descreve o comportamento
> a partir de **`yc-interpreter:amd64-260909b`**. Houve **duas** imagens no mesmo dia: na primeira
> (`amd64-260909`, sem o `b`) a entity com recorte alcançada pela **rota interna de cluster** era
> recusada com `403` — o que quebra qualquer processor que leia a projeção. **Não declare o recorte
> contra superfície que ainda esteja na imagem sem o `b`.**

### Como se faz — o percurso inteiro

O objetivo, dito em uma frase: *"quem tem o papel `OPERADOR` vê só a própria ficha; quem tem `ADMIN` vê
todas"*.

**1 · O modelo declara.** No `_conf` da entity, publicado pelo forger — ver
[forger — entity](../forger/endpoints/entity.md), que é onde a forma e as recusas estão descritas:

```jsonc
"_conf": {
  "accessControl": {
    "read":  ["MASTER", "ADMIN", "OPERADOR"],   // quem pode ler a entity
    "write": ["MASTER", "ADMIN"],
    "scope": {
      "read": {
        "OPERADOR": { "rows": { "by": "username" } }   // e OPERADOR, só as linhas dele
      }
    }
  }
}
```

`ADMIN` não aparece no `scope` — logo lê tudo. `OPERADOR` aparece — logo lê só o que é dele. O atributo
`username` da entity é o que amarra a linha ao titular; a próxima seção trata disso.

**2 · A consulta não muda.** O cliente envia o mesmo corpo de sempre. Não há nada a acrescentar ao
pedido, e nada que o desligue:

```jsonc
{ "pessoa": { } }        // "me devolva as pessoas"
```

**3 · O que volta depende de quem perguntou.** A mesma requisição, três chamadores:

| Quem chama | Recebe |
|---|---|
| `ana.silva`, papel `OPERADOR` | **uma** linha — aquela cujo `username` é `ana.silva` |
| `chefe`, papel `ADMIN` | **todas** as linhas |
| alguém sem papel em `read` | `403` — nunca chega ao recorte |

### O que torna uma linha "sua"

O recorte compara **o atributo declarado em `rows.by`** com **quem está chamando**. Hoje "quem está
chamando" é o **`username` do token** — então o atributo precisa conter esse mesmo valor.

Três consequências práticas, e a terceira é a que mais morde:

- **o atributo é escolha de quem modela.** Pode chamar-se `username`, `login`, `titular`: o que importa
  é o **valor** ser o do token, não o nome da coluna;
- **metadado da plataforma não serve** — `id`, `loguser`, `logrole`, `logversion` e `logdate` são
  recusados. `loguser` registra **quem escreveu** a linha, não **de quem** ela é: ficha cadastrada por um
  administrador levaria o login dele, e o recorte devolveria zero linhas ao dono da ficha;
- ⚠️ **atributo vazio = ficha invisível para o próprio dono.** Se a linha existe e o campo não foi
  preenchido, ela não casa com ninguém — e a resposta é `200` com lista vazia, **indistinguível de "não
  existe"**. Quando a ficha é criada por outra pessoa, garantir que esse campo seja preenchido é parte da
  modelagem, não detalhe de implementação.

> **Está depurando uma lista vazia que não deveria estar vazia?** Esta e a condição irmã — o atributo
> conter valor diferente do que identifica o usuário no token — **falham fechado**: `200`, lista vazia,
> e **nenhum erro para ninguém**, nem para quem consulta nem para quem publicou a entity. Nenhuma das
> duas é verificada na publicação: o forger confere que o atributo **existe**, não o que ele **contém**.
> Se um titular relata que "sumiu tudo", **verifique estas duas antes de qualquer outra coisa** —
> detalhe e diagnóstico em
> [forger — o que o `rows.by` precisa ser](../forger/endpoints/entity.md#recorte-de-leitura-por-titular-accesscontrolscope).

### `read` e `scope`, conjugados

A regra que explica todo o resto: **`scope` nunca concede acesso, só estreita o que o `read` já
permitiu.**

| Situação do papel | Resultado |
|---|---|
| **fora** de `accessControl.read` | `403` — nunca chega ao recorte |
| em `read`, **fora** de `scope.read` | lê a tabela inteira |
| em `read`, **dentro** de `scope.read` | lê só as linhas dele |
| **vários papéis**, um deles fora do `scope` | lê tudo — o menos restritivo vence |

A última linha decorre das outras, e não é exceção: se um dos papéis já autorizava a tabela inteira, o
recorte de outro papel não tem o que tirar.

### Quando quem consulta é um processor, e não uma pessoa

Processor do br-service que lê a projeção **não** é usuário, e a distinção decide o que ele recebe:

| O processor precisa ver… | Use |
|---|---|
| só o que é do usuário que disparou o comando | o `authToken` do corpo, no endpoint seguro — **o recorte se aplica** |
| **além** do usuário — validar contra dado de terceiro | o **endpoint interno de cluster**, que não tem usuário e **não recorta** |

No caminho **assíncrono** não há escolha: não há JWT, e o endpoint interno é o único caminho. Detalhe
do contrato em [br-service](../br-service/README.md).

O endpoint interno **não recortar** vale a partir de `amd64-260909b` — antes dela ele era recusado com
`403`, que é a razão de a versão importar aqui mais que em qualquer outra seção deste documento.

> ⚠️ Usar o token do usuário para uma regra que precisa enxergar dado de outra pessoa devolve **lista
> vazia com `200`** — não um erro. É o engano mais fácil de cometer aqui.

### O que muda na consulta

- **Quem não declara não muda.** Entity sem a declaração responde exatamente como antes.
- **É por papel, na mesma entity.** Um papel pode ler a tabela inteira e outro só as linhas dele.
- **Vários papéis: o menos restritivo vence.** Se qualquer papel autorizado do usuário está fora do
  recorte, não há recorte — quem acumula um papel amplo não é cortado pelo papel menor.
  **Acumular papéis sempre amplia, nunca restringe**, e isso é decisão de desenho, não efeito colateral:
  sem ela, quem tem um papel administrativo perderia o alcance dele por também ter um papel comum. A
  consequência a considerar ao atribuir papéis: a pessoa que acumula um papel recortado e um não
  recortado **na mesma entity** lê pelo não recortado.
- **O recorte não negocia com o seu critério.** O que você envia fica isolado, e o recorte entra por
  "E": `_connective: "OR"` **não** o transforma em alternativa, e um `ilike` amplo continua recortado.
- **`_count` conta o conjunto já recortado**, não a tabela.
- **`204` continua significando "nenhum resultado".** Sob recorte, lista vazia pode querer dizer
  "nenhuma linha é sua" — não é erro, e não há como distinguir os dois casos pela resposta.
- **Recorte em entity alcançada por associação muda o conjunto devolvido.** Se a entity **associada**
  é a que declara o recorte, o filtro dela entra no `WHERE` — não no `ON` do join. Consequência: a linha
  principal cuja associação **não é sua** (ou é nula) **não aparece**, em vez de aparecer com a
  associação vazia. Não é vazamento — é mais restritivo —, mas quem espera "o associado aparece, o
  professor vem em branco" recebe outra coisa.

Quem **declara** a chave é o forger, no `_conf` da entity — ver
[forger — entity](../forger/endpoints/entity.md). Aqui só se descreve o efeito na **consulta**.

## Pontos de coordenação
- **CP-2** — consulta as projeções (tabelas) criadas pelo forger.
- **CP-5** — lê o que o fluxo es-n → projeção mantém atualizado.
Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Enviar **sempre** `Authorization` **e** o cabeçalho de tenant.
- [ ] Usar **rótulos únicos** por critério no modo array (rótulos repetidos misturam resultados).
- [ ] Não inventar predicados — usar o vocabulário provisionado para o tenant.
- [ ] Tolerar **atraso de propagação**: logo após um comando, a projeção pode ainda não refletir a
      mudança (atualização assíncrona).
- [ ] Tratar `204` como "nenhum resultado", não como erro.
- [ ] Sob **recorte por titular**, não concluir "a tabela está vazia" a partir de uma lista vazia — ela
      pode significar apenas "nenhuma linha é sua".
