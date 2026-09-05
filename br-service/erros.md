# br-service · catálogo de erros

> Envelope e códigos. Guia: [README.md](README.md).

## Envelope de erro

```json
{ "status": "error", "mensagem": "<descrição>", "tipo": "<tipo do erro>" }
```

> ⚠️ **Uma exceção, e ela quebra quem trata erro pelo campo `tipo`.** A falha de **validação do corpo**
> responde `400` com um corpo **diferente**: `{ "erro": "<descrição>" }` — um campo só, sem `status`, sem
> `mensagem` e sem `tipo`. Todos os demais `400` usam o envelope acima. Trate o `400` pelo código HTTP e
> só leia `tipo` depois de confirmar que o campo existe.

## Casos

| HTTP | Caso | Mensagem típica | Correção |
|---|---|---|---|
| `400` | Validação | **outro corpo** — `{ "erro": "..." }`, sem `status`/`mensagem`/`tipo`. Ocorre com corpo nulo, não-objeto ou sem `route`. | Enviar `{ "route": "...", "data": {...} }`. |
| `400` | Rota ausente | `route` não informado. | Incluir `route`. |
| `400` | Rota desconhecida | rota inválida — a resposta lista as rotas disponíveis. | Usar uma rota existente; conferir o nome no model. |
| `400` | Exceção do processador | mensagem e tipo do erro lançado pelo processador. | Corrigir os dados ou o processador. |

## Nota
O br-service responde sempre de forma **síncrona** ao persistence-crs. Como é um transformador puro,
não há estados intermediários a recuperar: em falha, o chamador (persistence-crs) trata o erro no
contexto do comando/coordenação.
