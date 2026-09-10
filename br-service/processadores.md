# br-service · autoria de processadores

> Como um processador é **escrito** e **publicado**, e o que faz uma rota declarada no model existir de
> fato em execução. **O que ele recebe e o que precisa devolver depende do contexto de invocação** —
> isso está em [contextos.md](contextos.md), e é leitura anterior a esta. O contrato visto pelo chamador
> está no [guia](README.md) e em [endpoints/br.md](endpoints/br.md).

## Contents
- Do arquivo à rota
- Forma do arquivo
- O que o processador recebe
- O que o processador devolve
- Erros
- Publicação
- Diagnóstico

---

## Do arquivo à rota

Um processador é um **arquivo de código**, e **o caminho do arquivo é a rota** — sem a extensão. Não há
registro, tabela de rotas nem declaração à parte: a árvore de pastas publicada *é* o mapa de rotas.

| Arquivo publicado | Rota resultante |
|---|---|
| `<org>/<project>/<bc>/<aggregate>/criar` | `<org>/<project>/<bc>/<aggregate>/criar` |
| `<org>/<project>/<bc>/<aggregate>/coordination/conta_from_pedido` | `<org>/<project>/<bc>/<aggregate>/coordination/conta_from_pedido` |

> Os caminhos acima aparecem **sem a extensão**, que é omitida em toda esta documentação. A extensão
> aceita é a mesma para todos os processadores e é informada no provisionamento — junto com os arquivos
> de dados de apoio, que a publicação também aceita. Errar a extensão é recusa na publicação, com
> mensagem: não é um caso que se descubra tarde.

É por isso que a [forma canônica da rota](README.md#forma-canônica-da-rota-obrigatória) é uma regra
sobre **como organizar as pastas**, e não uma convenção de nomenclatura. O **primeiro segmento é a
organização**: é ele que separa o espaço de uma organização do de outra, tanto no mapa de rotas quanto
no escopo da publicação.

Nomes iniciados por ponto são ignorados na varredura — uma pasta assim nunca vira rota.

## Forma do arquivo

**O arquivo tem de exportar uma função** — é essa função que a rota executa, e é a única coisa que o
carregador procura. Há duas formas aceitas, e as duas valem igual:

| Forma | Exigência |
|---|---|
| **exportação direta** — o módulo **é** a função | nenhuma; é a recomendada |
| **exportação nomeada** — o módulo é um objeto com uma função dentro | o nome exportado tem de ser **exatamente o nome do arquivo**, sem a extensão |

A função recebe os argumentos que o contexto de invocação determina e pode ser assíncrona; o resultado é
aguardado. Ver [contextos.md](contextos.md).

> **⚠️ O que não é função é descartado em SILÊNCIO.** Um arquivo que exporta um objeto, que erra o nome
> na exportação nomeada, ou que falha no instante em que é carregado **não vira rota e não produz erro
> visível** — ele simplesmente não aparece no mapa, e a chamada correspondente falha depois como "rota
> inexistente". Processador que "sumiu" é quase sempre isto.
>
> A contrapartida: **todo** arquivo publicado que exporte uma função vira rota executável. Um módulo
> auxiliar, compartilhado entre processadores e que não deve ser chamável, precisa exportar **outra
> coisa que não uma função** — um objeto, por exemplo — para não virar rota por acidente.

Republicar no mesmo caminho substitui a versão anterior: a chamada seguinte já executa o código novo,
sem versão antiga presa em memória.

## O que o processador recebe

**Não é o autor quem escolhe a assinatura — é o contexto de invocação.** O serviço monta a chamada a
partir dos campos que o corpo traz:

| O corpo traz | A chamada |
|---|---|
| `data` + `authToken` + `tenantIds` | `f(data, authToken, tenantIds)` |
| `data` + `authToken` | `f(data, authToken)` |
| `data` | `f(data)` |
| sem `data` | `f(corpoInteiro)` |

Na prática isso quer dizer **três argumentos no contexto síncrono e um só nos dois assíncronos** — e um
`authToken` nulo, que ocorre em toda operação sem usuário autenticado, também reduz a chamada a um
argumento. Escreva a função para o contexto em que ela vai rodar, e **nunca leia o segundo parâmetro sem
checar**: [contextos.md](contextos.md).

Não há JWT em contexto assíncrono — nunca, e não é "pode não haver": a credencial gravada no evento não
guarda o token. Como ler dados de volta sem ele, e com que consequência de acesso, está em
[acesso-a-dados.md](acesso-a-dados.md).

## O que o processador devolve

A função pode ser assíncrona; o resultado é aguardado. O que ela retorna vai ao persistence-crs **sem
envelope acrescentado pelo serviço** — mas **o que ela deve retornar não é o mesmo nos três contextos**:

| Contexto | Forma do retorno |
|---|---|
| regra de negócio | o objeto do comando, **direto** — mesclado por **whitelist**, e chave nova é descartada em silêncio |
| coordenação | `{"processedData": {"targetCommand": {…}}}` |
| projeção | `{"processedData": {"<projeção>": {"aggregateid": …}}}` |

Forma exata, campos obrigatórios e o que acontece quando se erra: [contextos.md](contextos.md). Contrato
de fio: [contrato de resposta](README.md#contrato-de-resposta).

## Erros

Lançar é o mecanismo previsto: a exceção vira `400` com `{ status, mensagem, tipo }`.

```
função(data):
    se !data.cliente: lança erro "Campo 'cliente' é obrigatório"
    retorna { ...data, cliente: data.cliente.trim() }
```

A mensagem e o tipo do erro lançado atravessam inteiros para o corpo da resposta:

```json
{ "status": "error", "mensagem": "Campo 'cliente' é obrigatório", "tipo": "Error" }
```

> Escrever a mensagem pensando em quem vai lê-la no `400` compensa: é o único texto do processador que
> chega ao chamador.

Catálogo completo: [erros.md](erros.md).

## Publicação

Processadores **não** acompanham o código do serviço: publicar uma regra de negócio não passa por
commit nem por reconstrução de imagem. A publicação envia um **pacote `.zip`** com a subárvore de uma
organização e é **escopada por essa organização** — cada envio substitui apenas o espaço da organização
declarada, nunca o de outra.

Exige duas credenciais, com papéis distintos: uma **autoriza a operação** e a outra (`Authorization:
Bearer <JWT>`) diz **quem é** e **a que organizações** pode publicar. Publicar em organização à qual o
portador não está vinculado é recusado.

A troca é atômica e a versão anterior fica guardada, de modo que uma publicação pode ser **desfeita**.
A resposta traz as rotas efetivamente **adicionadas e removidas** — e o serviço passa a atender as novas
rotas **sem reinício**.

## Diagnóstico

**Não conte com log**: a carga do mapa de rotas é silenciosa, e um arquivo recusado não aparece em lugar
nenhum. Os dois sinais confiáveis:

- **A resposta da publicação** — lista as rotas que realmente entraram. É a única fonte que distingue
  "arquivo enviado" de "rota carregada e atendível". Rota esperada que não aparece nessa lista é, quase
  sempre, export fora das duas formas aceitas.
- **Chamar a rota** — rota inexistente responde `400` **listando as rotas disponíveis**, o que confirma
  de uma vez se o processador entrou e sob que nome.

Quando o arquivo foi aceito mas a rota não apareceu, o teste local é importar o próprio arquivo e
conferir que o export é uma **função** (e não um objeto): o erro que a importação levantar aí é o mesmo
que a carga engoliu.
