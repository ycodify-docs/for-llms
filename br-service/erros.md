# br-service · catálogo de erros

> Envelope e códigos, e **o que cada contexto de invocação faz com a falha** — que não é o mesmo nos
> três. Guia: [README.md](README.md) · [contextos.md](contextos.md).

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

## ⚠️ O código que o serviço responde não é o que o chamador vê

O br-service responde ao **persistence-crs**, não ao cliente final. O que acontece com essa resposta
depende do contexto:

| Contexto | O que o chamador faz com o `400` | O que o cliente final vê |
|---|---|---|
| **regra de negócio** (síncrono) | aborta o comando: nenhum evento é gravado | **`510`** — não `400`. A mensagem do processador viaja no corpo dessa resposta |
| **coordenação** (assíncrono) | devolve a mensagem à fila | nada: o comando de origem já respondeu `200` |
| **projeção** (assíncrono) | devolve a mensagem à fila | nada, pelo mesmo motivo |

**Quem trata `400` no cliente nunca encontra a mensagem do processador.** A recusa por papel do comando
é a exceção que confirma a regra: essa chega como `403`, com mensagem própria, e não passa pelo
br-service — é avaliada **antes**.

## A falha que não produz erro nenhum

Nem toda falha de processador é exceção. Devolver a forma errada responde `200` e o resultado é
descartado sem registro visível ao autor — envelope faltando no contexto assíncrono, chave nova no
síncrono, projeção sem `aggregateid`. Os três casos, e como reconhecê-los, estão em
[contextos.md](contextos.md).

## Nota
O br-service responde sempre de forma **síncrona** ao persistence-crs, e não guarda estado entre
chamadas: em falha, quem trata é o chamador, no contexto em que estava. Isso **não** quer dizer que o
processador seja puro — processadores de coordenação e de projeção normalmente **leem** dados para
enriquecer o mapeamento ([acesso-a-dados.md](acesso-a-dados.md)), e nos contextos assíncronos podem ter
efeito externo, desde que tolerem reentrega.
