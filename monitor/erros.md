# monitor — erros dos endpoints de relato

Todas as respostas de erro do monitor têm o formato `{ "error": "<código>", "details": "<motivo legível>" }`.
O campo `details` diz qual regra foi violada.

| HTTP | `error` | Quando acontece | O que fazer |
|---|---|---|---|
| 400 | `bad_request` | falta `X-Relator-Username`; cabeçalho do relator longo demais; JSON ilegível; `tipo` fora de Falha, Dúvida e Sugestão; nenhuma mensagem, mais de 20, texto vazio ou texto acima de 10.000 caracteres; `contexto` ou `diagnostico` que não são objeto; `estado` inválido no filtro da equipe | corrigir a requisição |
| 401 | `unauthorized` | lado do chamador: credencial de serviço ausente, errada ou não configurada no monitor. Lado da equipe: token ausente, inválido ou expirado | conferir a configuração do chamador com a infra; não repetir a chamada em laço |
| 403 | `forbidden` | só no lado da equipe: token sem `API_MASTER`/`API_ENGINEER` em organização ativa, ou `?org=` fora das organizações do portador | — |
| 404 | `not_found` | relato inexistente **ou** de outro usuário ou organização. Os dois casos são indistinguíveis de propósito | conferir o id e o usuário do cabeçalho |
| 409 | `conflict` | mensagem em relato fechado, ou fechamento de relato já fechado | o usuário abre um relato novo |
| 413 | `payload_too_large` | corpo acima de 256 KB, `diagnostico` acima de 64 KB ou `contexto` acima de 4 KB | a casca já corta o diagnóstico no teto; conferir o que foi acrescentado a ele |

> **Nem todo erro destas rotas vem do monitor.** Um `404` sem esse corpo e com cabeçalhos da borda vem da
> **borda**, que não tem a rota ([gateway/erros.md](../gateway/erros.md#404--a-borda-não-reconheceu-a-rota)).
> O mesmo vale para o `413` do teto da borda e para o `401` de quem chama sem o cabeçalho de identificação
> dela ([gateway/erros.md](../gateway/erros.md#401--falta-o-cabeçalho-de-identificação)).
