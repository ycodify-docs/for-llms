# br-service · endpoints · consultar o log de execução

> Devolve o **registro das execuções** de regra/coordenação, filtrado por termo e por intervalo de tempo.
> Autorizado por **chave própria**, não pelo token de usuário. Guia: [../README.md](../README.md).

## Requisição

Duas formas — a segunda acrescenta o filtro por arquivo:

```
GET /v3/brservice/logs/query/{term}/from/{f}/to/{t}
GET /v3/brservice/logs/query/{term}/file/{file}/from/{f}/to/{t}
```

| Parâmetro | O que é |
|---|---|
| `term` | texto procurado no registro. Casa por **substring**, sem diferenciar maiúsculas. `*` devolve tudo do intervalo |
| `file` | trecho do campo `erroArquivo` — **onde o erro foi lançado**. Casa por substring. `*` aceita qualquer arquivo |
| `f` · `t` | limites do intervalo, **inclusivos**. ISO 8601 (`2026-08-31T00:00:00Z`) ou epoch em **milissegundos** |

> **`term` e `file` somam, não substituem.** Os dois filtros são aplicados juntos: o registro precisa
> conter o termo **e** ter sido lançado no arquivo indicado. Para filtrar só por arquivo, use `*` no `term`.

> **Não use barra em `file`.** O filtro casa por substring justamente para isso: `atualizar` ou
> `associado` bastam para achar o caminho inteiro do arquivo. Barra
> codificada (`%2F`) dentro de um segmento de caminho costuma ser recusada pelos proxies do caminho
> público, antes mesmo de chegar ao serviço.

> **Filtrar por arquivo implica erro.** Execução bem-sucedida tem `erroArquivo` vazio e não casa nenhum
> `file` além de `*`.

Os parâmetros são parte do caminho — codifique-os quando contiverem caractere reservado.

### Cabeçalho

| Cabeçalho | Obrigatório | Valor |
|---|---|---|
| `X-Logs-Key` | sim | a chave de consulta, no formato UUID, entregue no provisionamento |

Esta rota **não** usa `Authorization` nem `X-Tenant-Id`: a chave é a credencial inteira.

## Resposta

`200` — objeto com o resultado da busca:

```json
{
  "total": 2,
  "truncado": false,
  "limite": 1000,
  "registros": [
    {
      "ts": "2026-08-31T02:10:52.494Z",
      "traceId": "<id de correlação, quando propagado>",
      "spanId": null,
      "tenantId": "<tenant>",
      "route": "<org>/<project>/<bc>/<aggregate>/<função>",
      "outcome": "success",
      "durationMs": 4,
      "bytesIn": 46,
      "bytesOut": 100,
      "dataKeys": ["nome"],
      "erroTipo": null,
      "erroMensagem": null,
      "erroArquivo": ""
    }
  ]
}
```

> Ilustrativo — `<...>` é placeholder, não JSON literal.

| Campo | O que é |
|---|---|
| `total` | quantos registros vieram nesta resposta |
| `truncado` | `true` quando o teto foi atingido e há mais registros no intervalo — estreite o intervalo ou o termo |
| `limite` | teto de registros por consulta |
| `registros[].outcome` | `success` · `validation_error` · `processor_error` |
| `registros[].tenantId` | tenant da execução. **Sempre presente**; vem `""` quando a requisição não trouxe tenant em lugar nenhum |
| `registros[].dataKeys` | **nomes** dos campos que chegaram em `data` — nunca os valores |
| `registros[].erroMensagem` | mensagem do erro, **truncada**; o corte é sinalizado no fim do texto |
| `registros[].erroArquivo` | **onde** o erro foi lançado, no formato `caminho:linha` — aponta o arquivo da função que falhou. `""` quando não houve erro |

> **Como o tenant é encontrado.** Ele não chega sempre no mesmo lugar: em execução **síncrona** vem no
> cabeçalho da requisição ou junto do corpo; em **coordenação e projeção** vem dentro do próprio payload do
> evento. A consulta normaliza isso — o campo `tenantId` é preenchido a partir de qualquer uma dessas
> origens, e só fica vazio quando nenhuma delas trouxe o dado.

**O payload não é registrado.** Só metadados. Os dados do comando podem conter informação pessoal e
credencial, então não são gravados — logo não há como recuperá-los por aqui.

## Erros

| Código | Quando |
|---|---|
| `400` | `f` ou `t` fora de formato, ou `f` posterior a `t` |
| `401` | `X-Logs-Key` ausente ou incorreta |
| `503` | consulta de log não provisionada neste ambiente |

Corpo de erro: `{ status, mensagem, tipo }`. Catálogo geral: [../erros.md](../erros.md).

## Alcance e retenção

- Só há registro do que passou pelo **próprio br-service** — execuções de regra, coordenação e projeção.
  O que acontece no motor de comandos não aparece aqui.
- O registro é gravado **depois** de a requisição terminar, em qualquer desfecho: sucesso, falha de
  validação e exceção do processador entram igual.
- **Sem garantia de retenção indefinida.** O intervalo consultável depende da política de retenção do
  ambiente; consultas a datas antigas podem voltar vazias mesmo tendo havido execução.
- Correlação com o motor de comandos: quando o chamador propaga um identificador de rastreio, ele aparece
  em `traceId` — é a chave para juntar os dois lados.

## Exemplos

Tudo o que houve num dia:

```
GET /v3/brservice/logs/query/*/from/2026-08-31T00:00:00Z/to/2026-08-31T23:59:59Z
X-Logs-Key: <uuid>
```

Só o que falhou numa rota:

```
GET /v3/brservice/logs/query/processor_error/from/1756605600000/to/1756692000000
X-Logs-Key: <uuid>
```

Rastrear uma execução específica pelo identificador de correlação:

```
GET /v3/brservice/logs/query/<traceId>/from/2026-08-31T02:00:00Z/to/2026-08-31T03:00:00Z
X-Logs-Key: <uuid>
```

Tudo o que falhou **num arquivo**, no dia:

```
GET /v3/brservice/logs/query/*/file/atualizar/from/2026-08-31T00:00:00Z/to/2026-08-31T23:59:59Z
X-Logs-Key: <uuid>
```

Os dois filtros somados — falhas **daquele arquivo** cuja mensagem contém um texto:

```
GET /v3/brservice/logs/query/obrigat%C3%B3rio/file/atualizar/from/2026-08-31T00:00:00Z/to/2026-08-31T23:59:59Z
X-Logs-Key: <uuid>
```
