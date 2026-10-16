# monitor — endpoints de relato

> Base pela borda: **`/v3/monitor/suporte/v1`**. Quem chama é **um serviço com credencial própria**
> ([quais](../README.md#mais-de-um-chamador)), nunca o browser. Corpo e respostas em JSON (UTF-8). Todas as
> rotas exigem os cabeçalhos abaixo. Guia: [monitor](../README.md).

## Cabeçalhos

| Cabeçalho | Obrigatório | O que é |
|---|---|---|
| `X-Monitor-Client-Key` | sim | credencial de serviço do chamador, uma para cada chamador. A infra entrega o valor à configuração do chamador, e ele **nunca** passa por código, documento ou mensagem. O monitor guarda só um resumo criptográfico dela. |
| `X-Relator-Username` | sim | o usuário da sessão do chamador. É por ele que cada rota alcança **só os relatos daquele usuário**. |
| `X-Relator-Org` | não | o nome da organização do relator ([quem enxerga](../README.md#quem-na-equipe-enxerga-o-relato)) |
| `X-Relator-Tenant` | não | o tenant do relator |
| `X-Request-Id` | não | o id da chamada no chamador; fica gravado no relato e nas mensagens do usuário |

- credencial ausente ou inválida → `401`;
- `X-Relator-Username` ausente → `400`;
- corpo acima de **256 KB** → `413`, antes de qualquer leitura.

Pela borda, a chamada leva também o **cabeçalho de identificação da borda**, que nesta rota é
`X-Forger-Credential` ([borda](../../gateway/README.md#cabeçalhos-de-identificação--exatamente-um-nunca-dois)).
Sem ele a borda responde `401`, e a chamada não chega ao monitor
([gateway/erros.md](../../gateway/erros.md#401--falta-o-cabeçalho-de-identificação)). A borda aplica também um
teto próprio de corpo por serviço
([gateway/erros.md](../../gateway/erros.md#413--corpo-ou-cabeçalhos-grandes-demais)).

## POST /relatos — criar

A carga tem a mesma forma para qualquer chamador. Esta é a que a casca monta:

```jsonc
{
  "tipo": "Falha",                           // Falha | Dúvida | Sugestão — com ou sem acento, qualquer caixa
  "titulo": "…",                             // opcional; ausente ou vazio → 1ª linha da 1ª mensagem; corta em 80
  "contexto": { "tela": "Pedidos" },         // opcional; objeto rótulo → valor, até 4 KB
  "mensagens": [{ "texto": "…" }],           // 1 a 20; a 1ª é o relato; cada texto até 10.000 caracteres
  "diagnostico": { "formato": 1 }            // opcional; objeto de até 64 KB (forma: shell/seguranca.md)
}
```

O `formato` 1 é o diagnóstico da casca. Outro chamador manda outro inteiro em `formato`, com a carga que
quiser dentro do teto ([o que o monitor faz com ela](../README.md#o-que-o-monitor-faz-com-o-conteúdo)).

Resposta `201`:

```json
{ "id": 42, "estado": "ABERTO", "criadoEm": "2026-09-27T04:00:00Z" }
```

## GET /relatos — os relatos do usuário

Parâmetros: `limit` (padrão 50, máximo 200) e `offset`. A lista vem do relato atualizado mais recentemente
ao mais antigo.

Resposta `200`:

```json
{ "total": 1, "limit": 50, "offset": 0,
  "rows": [{ "id": 42, "tipo": "FALHA", "titulo": "…", "estado": "RESPONDIDO",
             "criadoEm": "…", "atualizadoEm": "…", "fechadoEm": null, "naoLido": true }] }
```

O `tipo` volta como `FALHA`, `DUVIDA` ou `SUGESTAO`.

## GET /relatos/{id} — um relato com a conversa

Resposta `200`: os campos da lista, mais `contexto` e
`mensagens: [{ "id", "autor": "USUARIO" | "EQUIPE", "texto", "em" }]` em ordem cronológica. **O diagnóstico
não vem.**

Relato de outro usuário → `404`, como se não existisse.

## POST /relatos/{id}/mensagens — o usuário responde

Corpo: `{ "texto": "…" }`.

Resposta `201`: `{ "id", "autor": "USUARIO", "em", "estado": "ABERTO" }`. Relato fechado → `409`.

## POST /relatos/{id}/lido — marcar como lido

Resposta `204`. Relato de outro usuário → `404`.

## GET /nao-lidos — o ponto de não lido

Resposta `200`: `{ "total": 2 }`, o número de relatos do usuário com resposta da equipe ainda não lida.

## Lado da equipe (tela do monitor)

A base é `/monitor/api/suporte`, com o token de acesso (`API_MASTER`/`API_ENGINEER`). `?org=` estreita a
consulta à organização escolhida, como no resto da tela do monitor.

| Rota | O que faz |
|---|---|
| `GET /relatos?estado=` | a fila, com `username`, `org`, `tenantId` e o `naoLido` da equipe |
| `GET /relatos/{id}` | o relato completo, com o diagnóstico e o chamador que o gravou (`cliente`) |
| `POST /relatos/{id}/mensagens` | resposta da equipe → `RESPONDIDO` |
| `POST /relatos/{id}/fechar` | `FECHADO`; fechar de novo → `409` |
| `POST /relatos/{id}/lido` | marca como lido pela equipe |
| `GET /nao-lidos` | quantos relatos têm mensagem do usuário ainda não lida |

Erros: [erros.md](../erros.md). Exemplos: [exemplos.md](../exemplos.md).
