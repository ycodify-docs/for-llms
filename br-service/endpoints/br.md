# br-service · endpoints · executar função (rota)

> Executa o **processador** identificado por uma **rota**. Chamado **pelo persistence-crs**, não pelo
> cliente. Guia: [../README.md](../README.md). O que muda de um contexto de invocação para outro:
> [../contextos.md](../contextos.md).

## Requisição

`POST /br`

Corpo:

```json
{
  "route": "<rota do processador>",
  "data": { "...": "..." },
  "authToken": "<token de acesso do usuário, ou null>",
  "tenantIds": ["<tenant>"]
}
```

> Ilustrativo — `<...>` é placeholder, não JSON literal.

| Campo | Obrigatório | O que é |
|---|---|---|
| `route` | **sim** | caminho textual do processador, na forma canônica, começando pela organização (ex.: `acme/vendas/vendas/pedido/validar`) |
| `data` | não | dados passados ao processador. **Ausente, o corpo inteiro é passado** no lugar |
| `authToken` | não | token de acesso do usuário. Só é enviado no **contexto síncrono**, e mesmo lá pode ser nulo |
| `tenantIds` | não | lista com **um** tenant, vindo do modelo. Só é enviada no contexto síncrono |

⚠️ **Estes quatro campos decidem quantos argumentos o processador recebe** — não é o autor quem escolhe:

| O corpo traz | O processador é chamado como |
|---|---|
| `data` + `authToken` + `tenantIds` | `f(data, authToken, tenantIds)` |
| `data` + `authToken` | `f(data, authToken)` |
| `data` | `f(data)` |
| sem `data` | `f(corpoInteiro)` |

Um `authToken` **nulo** conta como ausente. Nos dois contextos assíncronos ele nunca é enviado, então
lá o processador recebe **sempre um argumento só** — ver [../contextos.md](../contextos.md).

### Cabeçalhos

`X-Tenant-Id` acompanha a chamada **no contexto síncrono**. Nos contextos assíncronos **nenhum cabeçalho
de identificação é enviado**, e o tenant viaja dentro de `data`.

## Resposta

- `200` — o que o processador retornou, **sem envelope acrescentado pelo serviço**:

```json
{ "...": "..." }
```

⚠️ **O que o processador deve retornar não é o mesmo nos três contextos** — o síncrono espera o objeto
do comando direto; os assíncronos exigem o envelope `processedData`. Formas exatas em
[../contextos.md](../contextos.md).

- `400` — rota inexistente ou exceção no processador:

```json
{ "status": "error", "mensagem": "<descrição>", "tipo": "<tipo do erro>" }
```

- `400` — **falha de validação do corpo** (corpo nulo, não-objeto, ou sem `route`). Corpo **diferente**,
  com um campo só:

```json
{ "erro": "<descrição>" }
```

> ⚠️ **Este `400` não é o que o cliente final vê no contexto síncrono.** O motor de comandos embrulha a
> falha, e o cliente recebe `510`. Ver [../erros.md](../erros.md).

## Comportamento
Segue o [ciclo de vida](../README.md#ciclo-de-vida-de-uma-requisição): validação → roteamento →
execução do processador → resposta direta. O serviço em si não faz chamadas externas; **o processador
pode fazer** — ver [../acesso-a-dados.md](../acesso-a-dados.md).

## Erros
`400` em todos os casos de falha (validação, rota desconhecida — com lista de rotas disponíveis —,
exceção do processador), mas em **dois formatos de corpo diferentes**: a falha de validação devolve
`{ erro }` e todo o resto devolve `{ status, mensagem, tipo }`. Catálogo: [../erros.md](../erros.md).
