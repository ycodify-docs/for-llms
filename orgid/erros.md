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
| `204` | Sem conteúdo | **Só em leitura:** não encontrado; "existe" falso; lista vazia. | Tratar como ausência, não erro. |
| `400` | Requisição inválida | Campo obrigatório ausente; formato inválido. | Corrigir o corpo/parâmetros. |
| `404` | Não encontrado | **Em escrita:** a entidade alvo (ou uma referenciada) não existe, e **nada foi gravado**. | Conferir o alvo e as referências; a mensagem diz o que não aconteceu. |
| `403` | Acesso negado | Sem o **papel** exigido **na organização**; violação de posse. | Usar credencial com o papel correto na org. |
| `500` | Erro interno | Falha inesperada. | Retentar; se persistir, escalar. |
| `510` | Falha **não tratada** (catch-all) | Exceção não mapeada ao executar a operação — **não** é validação de campo (essas dão `400`/`204`). Ex.: corpo incompleto que não casa a forma exata; contexto que impede montar o objeto; falha em operação dependente (ex.: envio de e-mail). | Conferir o **corpo exato** e os **pré-requisitos** do endpoint; garantir que entidades referenciadas existem; então retentar. |

> **`510` na associação conta-papel (`/ua/account-role`):** quase sempre é **corpo fora da forma** ou
> **referência inexistente** — não um campo malformado. Garanta `account.username`, `role.name` **e**
> `role.owner` (obrigatório no `/ua`), que a conta e o papel (`name`+`owner`) **existam**, e que o corpo
> seja **objeto** (associar/status) ou **array** (lote) conforme o endpoint. Ver
> [endpoints/ua-associacao.md](endpoints/ua-associacao.md).

## Categorias (diagnóstico)

| Categoria | Significado |
|---|---|
| Autenticação | Token ausente/inválido em endpoint `/up`/`/ua`. |
| Autorização | Papel insuficiente ou posse de organização ausente (`403`). |
| Validação | Corpo/parâmetro malformado ou incompleto (`400`). |
| Não encontrado | Entidade inexistente: `404` em escrita (nada foi gravado), `204` em leitura (só ausência). |
| Serviço dependente | Falha ao consultar/persistir em outro serviço (forger/persistência). |
| Envio de e-mail | Falha na entrega de notificação (`510`). |

## Nota

`204` **não** é erro — é "não encontrado / vazio", e **só aparece em leitura**. Endpoints `/open/*` não
exigem autenticação; demais exigem `Authorization` e o **papel** adequado **na organização** alvo.

### ⚠️ Escrita que não encontrou o alvo responde `404`, não `204` (desde 2026-09-05)

Até essa data, **31 operações de escrita** sinalizavam "não encontrei" com `204`. `204` é família **2xx** —
sucesso sem corpo —, então:

```
gravou     → 200, corpo vazio
não gravou → 204, corpo vazio
```

Os dois eram 2xx e vazios; `response.ok` era `true` nos dois, e a mensagem de erro era descartada pelo
protocolo (`204` proíbe corpo). Escrita que falhava era **invisível por construção**: quem marcava um papel
como público recebia `204` e seguia achando que tinha configurado.

Hoje toda escrita que não acha o alvo devolve **`404` com mensagem que diz o efeito que não ocorreu** —
`"account not found: the password was not changed."`, `"O papel informado não existe: nada foi removido."`
Alcança `PUT`/`DELETE`/`POST` de `/ua/role`, `/ua/account*`, `/ua/account-role*`, `/up/account*`,
`/up/org` e `/up/account-role-org/replace-*`.

**Os dois casos mais perigosos que isso corrigiu:** `DELETE` de papel inexistente (num `DELETE`, `204` **é**
o código canônico de sucesso — a leitura era a oposta da correta) e troca de senha em conta inexistente (o
usuário recebia sucesso e a senha não mudava).

**Se você ainda vir `204` numa escrita, está falando com uma versão antiga do orgid.** Leitura segue em
`204`, e o `/exists` depende disso — ali `200`/`204` é o contrato, não um erro.
