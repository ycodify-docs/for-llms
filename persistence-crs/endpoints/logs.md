# persistence-crs · endpoints · consulta de logs

> Recupera os **registros de log** produzidos pelo próprio serviço, filtrados por **serviço** (`crs` ou `q`),
> **termo** e **recorte temporal**. Serve para diagnóstico de uma operação já executada — não é endpoint de
> domínio e **não** usa o token de acesso: a autorização é uma **chave própria**, em cabeçalho.
> O mesmo endpoint atende os dois lados da persistência; o serviço desejado vai no path.
> Guia: [../README.md](../README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` são placeholders, não valores literais.

## Requisição

```
GET /logs/service/{service}/query/{term}/from/{from}/to/{to}
```

| Item | Valor |
|---|---|
| Cabeçalhos | **`X-Logs-Key`** — a chave que autoriza a consulta (**não** é o `Authorization` do domínio) · **`X-Tenant-Id`** — **obrigatório**: delimita o resultado ao tenant |
| `{service}` | `crs` (lado de escrita) ou `q` (lado de leitura) — seleciona de quais registros se trata |
| `{term}` | texto a procurar no registro. Vale `*` para "qualquer" (pode ir literal). Só precisa de URL-encoding se contiver caractere que quebra o path — barra, espaço, `:` ou `%` |
| `{from}` / `{to}` | início e fim do recorte temporal (ver formatos aceitos abaixo) |

> **Isolamento por tenant.** O `X-Tenant-Id` **não é opcional**: a resposta traz **apenas** registros
> daquele tenant. É o que impede um chamador de enxergar operação de outro cliente — a chave de consulta
> sozinha não faz essa separação.

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

O endpoint vive nos dois ambientes; o que muda é o prefixo, como em toda a persistência
(ver [autenticação — URL base](../../06-autenticacao.md)):

| Ambiente | URL |
|---|---|
| **Teste** | `/v3/persistence/**t**/logs/service/{service}/query/{term}/from/{from}/to/{to}` |
| **Produção** | `/v3/persistence/logs/service/{service}/query/{term}/from/{from}/to/{to}` |

> **Disponibilidade:** o ambiente de **teste** está **publicado e em uso desde 2026-08-31**. A rota de
> **produção** segue o mesmo contrato, mas **ainda não está publicada** — e, quando estiver, devolverá bem
> menos: o nível de log em produção é mais restritivo que o de teste, então a maior parte dos registros de
> detalhe não existe lá.
>
> **A rota de produção ainda não está publicada.** Publicá-la não é só declarar o caminho: a borda da
> plataforma só encaminha caminhos **explicitamente liberados**, em mais de uma camada, e um caminho novo
> sob um prefixo já existente **também** precisa ser liberado. Enquanto isso não for feito, chamar a URL de
> produção devolve `403` — e esse `403` vem da borda, não deste endpoint (ver a tabela em `## Erros`).

### Exemplo (ambiente de teste)

Substitua `<chave>` e `<tenant>` pelos valores do seu ambiente. Note: **só** `X-Tenant-Id`, sem credencial.

```bash
curl -s "https://api.ycodify.com/v3/persistence/t/logs/service/crs/query/*/from/2026-08-31T00:00:00Z/to/2026-08-31T23:59:59Z" \
  -H "X-Tenant-Id: <tenant>" \
  -H "X-Logs-Key: <chave>"
```

Respostas que se deve esperar deste comando, e o que cada uma significa:

| Resposta | Leitura |
|---|---|
| `200` + corpo | há registros no recorte |
| `204` sem corpo | consulta válida, nada casou — ver a nota abaixo antes de suspeitar de falha |
| `403` | chave ausente ou errada |
| `400` **com corpo** | algo no path: serviço inválido, data irreconhecível, `to` antes de `from`, tenant malformado |
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
- **Registro sem tenant não é devolvido.** Nem todo registro carrega a marca do tenant — os produzidos fora
  de uma requisição (processamento de fila da projeção, inicialização do serviço, mensagens de framework)
  não a têm. Esses **ficam de fora**, por segurança: é preferível omitir a arriscar devolver operação de
  outro cliente. Consequência prática: **o que acontece no caminho assíncrono da projeção não aparece nesta
  consulta**.
- A consulta é **somente leitura** e não altera estado.
- O alcance é o do **próprio serviço**: registros anteriores ao início do serviço em execução podem não estar
  disponíveis.

## Erros

| Código | Quando |
|---|---|
| `400` | `X-Tenant-Id` ausente ou malformado; `{service}` diferente de `crs`/`q`; data em formato não reconhecido; `to` anterior a `from` |
| `403` | `X-Logs-Key` ausente ou inválida |
| `503` | o serviço não tem registros acessíveis para consultar |

**Respostas que não vêm deste endpoint** — úteis para não procurar defeito no lugar errado:

| Código | De onde vem | Quando |
|---|---|---|
| `400` **com corpo vazio** | gateway | `X-Tenant-Id` e `X-Forger-Credential` enviados juntos (ver aviso no topo). O `400` deste endpoint sempre traz corpo. |
| `403` **com `{"error":"Forbidden","message":"Access denied."}`** | borda da plataforma | o caminho não está publicado no ambiente consultado (típico ao tentar a URL de produção antes de ela existir) |
| `404` | fallback do gateway | a rota não casou — caminho digitado errado, ou ambiente sem a rota |

Catálogo: [../erros.md](../erros.md).

> **A chave é secreta.** Não a coloque em repositório, em exemplo copiado nem em log de cliente. Sem a chave
> configurada no serviço, o endpoint **nega todas** as chamadas — não existe modo aberto por omissão.
