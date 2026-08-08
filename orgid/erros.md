# orgid · catálogo de erros

> Envelope e códigos. Guia: [README.md](README.md).

## Envelope
Em erro, a resposta tem o status HTTP e uma mensagem. (Internamente as mensagens seguem o padrão
`CÓDIGO:descrição`.)

## Códigos HTTP

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `200` | OK | Leitura/atualização/remoção bem-sucedida; "existe" verdadeiro. | — |
| `201` | Criado | Criação de conta/vínculo. | — |
| `204` | Sem conteúdo | Não encontrado; "existe" falso; lista vazia. **Em `POST /open/ua/account-role`: papel inexistente — e nada é criado.** | Tratar como ausência, não erro. |
| `400` | Requisição inválida | Campo obrigatório ausente; formato inválido; recurso referenciado inexistente. | Corrigir o corpo/parâmetros. |
| `403` | Acesso negado | Sem o **papel** exigido **na organização**; violação de posse. | Usar credencial com o papel correto na org. |
| `409` | Conflito | Identificador **único** já em uso. No registro público `POST /open/ua/account-role`, `account.username` já existe **em toda a plataforma** — o login é único globalmente, **não** por organização. | Escolher outro `username`; checar antes com `GET /open/{up\|ua}/account/by/username/{username}/exists`. |
| `500` | Erro interno | Falha inesperada. | Retentar; se persistir, escalar. |
| `510` | Falha **não tratada** (catch-all) | Exceção não mapeada ao executar a operação — **não** é validação de campo (essas dão `400`/`204`). Ex.: corpo incompleto que não casa a forma exata; contexto que impede montar o objeto; falha em operação dependente (ex.: envio de e-mail). | Conferir o **corpo exato** e os **pré-requisitos** do endpoint; garantir que entidades referenciadas existem; então retentar. |

> **`510` na associação conta-papel (`/ua/account-role`):** quase sempre é **corpo fora da forma** ou
> **referência inexistente** — não um campo malformado. Garanta `account.username`, `role.name` **e**
> `role.owner` (obrigatório no `/ua`), que a conta e o papel (`name`+`owner`) **existam**, e que o corpo
> seja **objeto** (associar/status) ou **array** (lote) conforme o endpoint. Ver
> [endpoints/ua-associacao.md](endpoints/ua-associacao.md).

> **`204` no registro público `/open/ua/account-role` é falha, não sucesso.** Se o papel (`name`+`owner`)
> não existir, a resposta é `204` **sem corpo** e **nada** é criado — nem a conta, nem o vínculo (a
> transação inteira é revertida). Só `200` confirma o registro. Ver
> [endpoints/publico.md](endpoints/publico.md).

## Categorias (diagnóstico)

| Categoria | Significado |
|---|---|
| Autenticação | Token ausente/inválido em endpoint `/up`/`/ua`. |
| Autorização | Papel insuficiente ou posse de organização ausente (`403`). |
| Validação | Corpo/parâmetro malformado ou incompleto (`400`). |
| Não encontrado | Entidade inexistente (`204`/`400`). |
| Conflito | Identificador único já em uso (`409`) — ex.: `username` no registro público. |
| Serviço dependente | Falha ao consultar/persistir em outro serviço (forger/persistência). |
| Envio de e-mail | Falha na entrega de notificação (`510`). |

## Nota
`204` **não** é erro — é "não encontrado / vazio". Endpoints `/open/*` não exigem autenticação; demais
exigem `Authorization` e o **papel** adequado **na organização** alvo.
