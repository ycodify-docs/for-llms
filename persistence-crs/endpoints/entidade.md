# persistence-crs · endpoints · entidade (escrita direta de linha)

> **Não há rota para escrever uma linha diretamente.** `POST /e` (via gateway, `POST /v3/persistence/c/e` e
> `POST /v3/persistence/t/c/e`) não existe e responde `404`. Guia: [../README.md](../README.md).

## O que usar no lugar

| Você quer… | Use |
|---|---|
| criar, alterar ou encerrar um registro do domínio | [comando](comando.md) no agregado: gera o evento, passa pelo RBAC de comando e pelo processor `br`, e a projeção é escrita a partir do evento |
| remover as linhas que casam um critério | [remoção por predicado](remocao-por-predicado.md) (`POST /e/by`) |
| ler linhas | [persistence-q — consulta](../../persistence-q/endpoints/consulta.md) |

A linha de uma projeção é escrita só pelo caminho interno do serviço, a partir do evento despachado pelo
[es-n](../../es-n/README.md). Ver [escrita de projeção](projecao.md).
