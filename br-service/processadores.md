# br-service · autoria de processadores

> Como um processador é **escrito** e **publicado**, e o que faz uma rota declarada no model existir de
> fato em execução. O contrato visto pelo chamador está no [guia](README.md) e em
> [endpoints/br.md](endpoints/br.md).

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

Um processador é um **arquivo `.js`**, e **o caminho do arquivo é a rota** — sem o sufixo `.js`. Não há
registro, tabela de rotas nem declaração à parte: a árvore de pastas publicada *é* o mapa de rotas.

| Arquivo publicado | Rota resultante |
|---|---|
| `<org>/<project>/<bc>/<aggregate>/criar.js` | `<org>/<project>/<bc>/<aggregate>/criar` |
| `<org>/<project>/<bc>/<aggregate>/coordination/conta_from_pedido.js` | `<org>/<project>/<bc>/<aggregate>/coordination/conta_from_pedido` |

É por isso que a [forma canônica da rota](README.md#forma-canônica-da-rota-obrigatória) é uma regra
sobre **como organizar as pastas**, e não uma convenção de nomenclatura. O **primeiro segmento é a
organização**: é ele que separa o espaço de uma organização do de outra, tanto no mapa de rotas quanto
no escopo da publicação.

Nomes iniciados por ponto são ignorados na varredura — uma pasta assim nunca vira rota.

## Forma do arquivo

O arquivo precisa exportar uma **função**, de uma das duas formas:

```javascript
// Forma 1 — export direto (recomendada)
module.exports = async (data) => { /* ... */ };

// Forma 2 — export nomeado, com o nome do ARQUIVO
const criar = async (data) => { /* ... */ };
module.exports = { criar };   // só funciona em um arquivo chamado criar.js
```

> **⚠️ O que não é função é descartado em SILÊNCIO.** Um arquivo que exporta um objeto, que erra o nome
> no export nomeado, ou que lança durante a importação **não vira rota e não produz erro visível** — ele
> simplesmente não aparece no mapa, e a chamada correspondente falha depois como "rota inexistente".
> Processador que "sumiu" é quase sempre isto.
>
> A contrapartida: **todo** arquivo publicado que exporte uma função vira rota executável. Um módulo
> auxiliar compartilhado entre processadores deve exportar um **objeto**, para não virar rota por
> acidente.

Republicar no mesmo caminho substitui a versão anterior: a chamada seguinte já executa o código novo,
sem versão antiga presa em memória.

## O que o processador recebe

A quantidade de argumentos depende do que o chamador envia no corpo:

| Corpo | Chamada |
|---|---|
| `data` + `authToken` + `tenantIds` | `fn(data, authToken, tenantIds)` |
| `data` + `authToken` | `fn(data, authToken)` |
| `data` | `fn(data)` |
| sem `data` | `fn(corpoInteiro)` |

O `authToken` é o JWT do usuário e **só chega no hook síncrono** (regra de negócio). Nos hooks
assíncronos (coordenação, projeção) não há JWT — a regra de acesso de volta a persistence-q/crs nesse
caso está no [guia](README.md#callback-do-processor--persistence-q--persistence-crs-endpoint-interno-de-cluster).

## O que o processador devolve

A função pode ser `async`; o resultado é aguardado. O que ela retorna vai **direto** ao persistence-crs,
sem envelope — e, na regra de negócio, é mesclado no comando por **whitelist**: chave nova retornada é
descartada em silêncio. Ver [contrato de resposta](README.md#contrato-de-resposta) e as três categorias
de retorno em [três categorias de processador](README.md#três-categorias-de-processador).

## Erros

Lançar é o mecanismo previsto: a exceção vira `400` com `{ status, mensagem, tipo }`.

```javascript
module.exports = async (data) => {
  if (!data.cliente) throw new Error("Campo 'cliente' é obrigatório");
  return { ...data, cliente: data.cliente.trim() };
};
```

```json
{ "status": "error", "mensagem": "Campo 'cliente' é obrigatório", "tipo": "Error" }
```

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
