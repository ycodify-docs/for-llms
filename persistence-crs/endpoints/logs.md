# persistence-crs · endpoints · consulta de logs

> Recupera os **registros de log** produzidos pelo próprio serviço, filtrados por **serviço** (`crs` ou `q`),
> **termo** e **recorte temporal**. Serve para diagnóstico de uma operação já executada — não é endpoint de
> domínio e **não** usa o token de acesso: a autorização é o **segredo do seu tenant**, enviado no corpo.
> O mesmo endpoint atende os dois lados da persistência; o serviço desejado vai no path.
> Guia: [../README.md](../README.md).

> **É `POST`, e a operação é de leitura.** O verbo não indica mudança de estado: ele existe porque a
> requisição leva um **corpo**, e é no corpo que vai o segredo — em caminho de URL ele apareceria em
> registro de acesso, em intermediário de rede e no histórico do cliente.

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` são placeholders, não valores literais.

## Requisição

```
POST /logs/service/{service}/query/{term}/from/{from}/to/{to}
```

| Item | Valor |
|---|---|
| Cabeçalhos | **`X-Tenant-Id`** — **obrigatório**: de qual tenant se quer o log. `Content-Type: application/json`. **Não** use o `Authorization` do domínio: este endpoint não o consulta |
| Corpo | **obrigatório**: `{ "secret": "<segredo do seu tenant>" }` |
| `{service}` | `crs` (lado de escrita) ou `q` (lado de leitura) — seleciona de quais registros se trata |
| `{term}` | texto a procurar no registro. Vale `*` para "qualquer". Só precisa de URL-encoding se contiver caractere que quebra o path — barra, espaço, `:` ou `%` |
| `{from}` / `{to}` | início e fim do recorte temporal (ver formatos aceitos abaixo) |

### Autorização — os dois valores, conferidos um contra o outro

**Nenhum dos dois autoriza sozinho.** O `X-Tenant-Id` diz de qual tenant se quer o log; o `secret` do
corpo é o **segredo daquele tenant** — o mesmo que o identifica no provisionamento; peça-o a quem
administra a sua organização. O serviço confere se **o par é o que está registrado**.

Consequência, e é o ponto: **apresentar o segredo de um tenant pedindo o log de outro é `403`.** Não
basta o segredo ser válido, nem o tenant existir. Trocar o `X-Tenant-Id` deixou de ser caminho para o log
alheio.

Existe ainda uma **chave de operação** da plataforma, que alcança qualquer tenant e serve ao suporte
diagnosticar de fora; ela é a exceção à regra do par, e continua exigindo o `X-Tenant-Id` para saber qual
log ler.

Corpo sem `secret` é **`400`** (erro de payload), não `403` — quem errou o corpo precisa saber disso.
Segredo que não confere é `403`. Não existe modo aberto por omissão.

> ⚠️ **Mande `X-Tenant-Id` sozinho — nunca junto com `X-Forger-Credential`.** O gateway recusa qualquer
> requisição que traga os dois cabeçalhos ao mesmo tempo, com **`400`** e corpo vazio, **antes** de chegar
> a este serviço (violação `multiple_headers`). Como o caminho fica sob `/v3/persistence/`, ele é
> rota-de-engine: identifica-se pelo tenant, não pela credencial. É um erro fácil de cometer, porque a
> credencial é o que se usa em outras rotas da plataforma — e o `400` resultante **não** vem deste
> endpoint, então não corresponde a nenhuma linha da tabela de erros abaixo.

**Formatos aceitos em `{from}`/`{to}`** (os dois podem usar formatos diferentes):

| Forma | Exemplo |
|---|---|
| epoch em milissegundos | `1756607400000` |
| data-hora com fuso | `2026-08-31T01:30:00.000-03:00` |
| data-hora sem fuso (assume o fuso do serviço) | `2026-08-31T01:30:00.000` |
| só a data (início do dia) | `2026-08-31` |

### URL externa — teste e produção

O endpoint vive nos dois ambientes, e em **ambos ele fica sob o prefixo `c`** — o mesmo do comando. Não
existe prefixo `logs` próprio na borda: o que a borda encaminha é `.../c/**`, e ela remove esse prefixo
inteiro antes de entregar ao serviço, de modo que o `logs/...` do fim do caminho é o que o serviço recebe.
(ver [autenticação — URL base](../../06-autenticacao.md)):

| Ambiente | URL |
|---|---|
| **Teste** | `/v3/persistence/t/c/logs/service/{service}/query/{term}/from/{from}/to/{to}` |
| **Produção** | `/v3/persistence/c/logs/service/{service}/query/{term}/from/{from}/to/{to}` |

> ⚠️ **O `c` não é opcional.** Sem ele — `/v3/persistence/logs/...` ou `/v3/persistence/t/logs/...` — o
> caminho não casa nenhuma rota da borda e a resposta é `404` do gateway, não deste endpoint. Versões
> anteriores desta página publicavam a URL sem o `c`; estavam erradas.

> **Disponibilidade:** os dois ambientes estão publicados — a rota de produção **existe** e não precisa de
> liberação nova, porque não é caminho novo na borda: é o prefixo `c`, que já era encaminhado. O que muda
> entre eles é o volume: o nível de log em produção é mais restritivo que o de teste, então a maior parte
> dos registros de detalhe não existe lá.

### Exemplo (ambiente de teste)

Substitua `<segredo>` e `<tenant>` pelos valores do seu ambiente. Note: **só** `X-Tenant-Id`, sem
credencial de domínio.

```bash
curl -s -X POST "<base>/v3/persistence/t/c/logs/service/crs/query/*/from/2026-08-31T00:00:00Z/to/2026-08-31T23:59:59Z" \
  -H "X-Tenant-Id: <tenant>" \
  -H "Content-Type: application/json" \
  -d '{"secret":"<segredo>"}'
```

Respostas que se deve esperar deste comando, e o que cada uma significa:

| Resposta | Leitura |
|---|---|
| `200` + corpo | há registros no recorte |
| `204` sem corpo | consulta válida, nada casou — ver a nota abaixo antes de suspeitar de falha |
| `403` | o segredo não é o daquele tenant |
| `400` **com corpo** | corpo sem `secret`; ou algo no path: serviço inválido, data irreconhecível, `to` antes de `from`, tenant malformado |
| `400` **sem corpo** | mandou credencial junto com o tenant — tire o `X-Forger-Credential` |

## Resposta

`200` — objeto com o recorte e os registros encontrados:

```json
{
  "service":  "crs",
  "tenantId": "<tenant consultado>",
  "term":     "<termo procurado>",
  "from":     "<início do recorte>",
  "to":       "<fim do recorte>",
  "matched":  128,
  "returned": 128,
  "truncated": false,
  "records": [ "<registro>", "<registro>" ]
}
```

- **`matched`** — quantos registros casaram o filtro.
- **`returned`** — quantos vieram nesta resposta.
- **`truncated`** — `true` quando `matched` passou do teto por resposta; refine o recorte ou o termo.

`204` — nenhum registro casou o filtro (não é erro).

> **Serviço recém-implantado devolve `204` com frequência**, e isso não indica defeito: logo após um
> reinício só existem registros de inicialização, que **não** carregam tenant e por isso ficam de fora (ver
> `## Comportamento`). Para ver resultado, consulte um recorte que contenha atividade real — uma operação
> autenticada de `crs`/`q` — e lembre que o nível de log do ambiente limita o que chega a ser gravado.

## Comportamento

- Um **registro** pode ocupar várias linhas (por exemplo, quando traz o rastro de uma falha). O endpoint
  devolve o registro **inteiro** como um item, não linha a linha.
- O filtro é conjuntivo: o registro precisa ser **do tenant informado**, estar **dentro do recorte**,
  pertencer ao **serviço** pedido **e** conter o **termo**.
- **Registro sem tenant não é devolvido.** Nem todo registro carrega a marca do tenant — os produzidos
  fora de uma requisição (inicialização do serviço, mensagens de framework) não a têm. Esses **ficam de
  fora**, por segurança: é preferível omitir a arriscar devolver operação de outro cliente.
- **O caminho assíncrono da projeção APARECE.** O consumidor de fila carimba o tenant nos seus registros,
  então o que acontece depois do comando — a projeção sendo aplicada — é consultável por aqui. (Versões
  anteriores desta página diziam o contrário.)
- **O alcance da consulta termina na rotação do arquivo, não no início do serviço.** O arquivo de log é
  rotacionado por tamanho e por idade; um recorte temporal que caia antes da rotação não acha nada, mesmo
  que o serviço estivesse no ar naquele momento.
- A consulta é **somente leitura** e não altera estado.
- O alcance é o do **próprio serviço**: registros anteriores ao início do serviço em execução podem não estar
  disponíveis.

## Erros

| Código | Quando |
|---|---|
| `400` | corpo ausente, não-JSON, ou sem `secret`; `X-Tenant-Id` ausente ou malformado; `{service}` diferente de `crs`/`q`; data em formato não reconhecido; `to` anterior a `from` |
| `403` | o `secret` do corpo não é o segredo do tenant informado — inclusive quando é o segredo **de outro** tenant |
| `500` | o arquivo de log existe mas não pôde ser lido |
| `503` | o serviço não tem registros acessíveis para consultar |

**Respostas que não vêm deste endpoint** — úteis para não procurar defeito no lugar errado:

| Código | De onde vem | Quando |
|---|---|---|
| `400` **com corpo vazio** | gateway | `X-Tenant-Id` e `X-Forger-Credential` enviados juntos (ver aviso no topo). O `400` deste endpoint sempre traz corpo. |
| `403` **com `{"error":"Forbidden","message":"Access denied."}`** | borda da plataforma | o caminho não está publicado no ambiente consultado |
| `404` **do gateway** | borda da plataforma | faltou o `c` no caminho (`/v3/persistence/logs/...` em vez de `/v3/persistence/c/logs/...`) |
| `404` | fallback do gateway | a rota não casou — caminho digitado errado, ou ambiente sem a rota |

Catálogo: [../erros.md](../erros.md).

> **A chave é secreta.** Não a coloque em repositório, em exemplo copiado nem em log de cliente. Sem a chave
> configurada no serviço, o endpoint **nega todas** as chamadas — não existe modo aberto por omissão.
