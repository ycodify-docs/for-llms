# persistence-crs · endpoints · escrita de projeção

> **Não há rota externa para escrever numa projeção.** `POST /e` não existe e responde `404`. Guia:
> [../README.md](../README.md).

## Como a projeção é escrita

A linha de uma projeção é gravada por um caminho **interno ao serviço**, que não passa por requisição
externa: o [es-n](../../es-n/README.md) despacha o evento gravado pelo [comando](comando.md) e o serviço
aplica a projeção.

**Idempotência e ordenação são desse caminho.** A aplicação é deduplicada por evento e pode ser ordenada
(nenhuma, estrita ou por agregado), conforme o despacho do es-n.

Para mudar uma linha de projeção, emita o [comando](comando.md) correspondente no agregado. Para remover
linhas por critério, use [remoção por predicado](remocao-por-predicado.md).
