# persistence-crs · endpoints · entidade (CRUD convencional)

> CRUD **direto** de linhas numa **entidade convencional** — uma tabela do domínio que **não** é
> projeção de um agregado event-sourced. Escreve/atualiza/remove o registro em uma chamada, sem comando
> nem evento. Para agregados (event-sourced), use [comando](comando.md); para consultar, use
> [persistence-q](../../persistence-q/README.md).
> Pré-requisitos: [conceitos](../../02-conceitos.md), [autenticação](../../06-autenticacao.md),
> a entity precisa existir no esquema do tenant ([forger/entity](../../forger/endpoints/entity.md)).
> Guia: [../README.md](../README.md).

> **URL externa — o segmento `t` escolhe o ambiente.** `POST /e` é o path **downstream**; via gateway:
> **produção** `POST /v3/persistence/c/e` · **teste** `POST /v3/persistence/t/c/e`. Ver
> [autenticação](../../06-autenticacao.md).

## Quando usar

Duas formas de escrever no domínio:

| Você quer… | Use | Como identifica a linha |
|---|---|---|
| Gravar um **agregado** (máquina de estados, histórico de eventos) | `POST /a/...` ([comando](comando.md)) | `aggregateid` (UUID) |
| Gravar uma **entidade convencional** (tabela simples, sem event-sourcing) | `POST /e` (este doc) | `id` (chave primária) |

Uma **entidade convencional** é modelada como qualquer entity ([forger/entity](../../forger/endpoints/entity.md)),
porém **sem** as colunas de projeção de agregado (`aggregateid`, `status`, os `whenAttribute` de evento) —
elas só são obrigatórias quando a entity **é** projeção de um agregado.

> **⚠️ `/e` não recusa nada — nem projeção de agregado, nem linha de outra pessoa.** O controle de
> acesso do `/e` é inteiro: (a) o token pertence ao tenant; (b) o papel do usuário está na lista
> `_conf.accessControl.write` da entity. **Não há escopo por linha:** a checagem pergunta apenas *"este
> usuário tem papel que escreve nesta tabela?"*, nunca *"esta linha é dele?"* — o `id` da linha alvo não
> chega à função que autoriza. Quem tem o papel escreve em **qualquer** linha, inclusive na de outro
> usuário do mesmo papel.
>
> O `/e` **não é comando**: não consulta `command.roles` e **não chama o processor `br`** — onde vivem as
> regras de escopo de dono ("só o titular altera seu registro"). Numa entity que **é** projeção de
> agregado soma-se o dano ao event sourcing: a linha muda **sem evento**, projeção e log de eventos
> divergem em silêncio, e um replay desfaz a alteração sem aviso.
>
> Consequência prática: `/e` é escrita administrativa. Exponha-o só a quem você exporia o banco — a
> plataforma não tem como distinguir o uso legítimo do abusivo. Ver [escrita de projeção](projecao.md).

## Requisição

`POST /e`

Cabeçalhos: `Authorization` + cabeçalho de tenant `X-Tenant-Id`; `Content-Type: application/json`.

> ⚠️ Os blocos abaixo são **ilustrativos**: `<...>` são placeholders, não JSON literal pronto para envio.

Corpo (operação única) — `data` é **sempre um array** de linhas:

```json
{ "action": "CREATE | UPDATE | DELETE", "data": [ { "<atributo>": "<valor>" } ] }
```

Corpo (lote) — um **array** de operações no formato acima, aplicadas na ordem:

```json
[
  { "action": "CREATE", "data": [ { "<atributo>": "<valor>" } ] },
  { "action": "UPDATE", "data": [ { "id": "<pk>", "<atributo>": "<novo valor>" } ] },
  { "action": "DELETE", "data": [ { "id": "<pk>" } ] }
]
```

## Operações

O verbo é escolhido pelo campo `action` (não pelo método HTTP — sempre `POST`):

- **CREATE** — insere as linhas de `data`. A chave primária **`id`** é **gerada pela plataforma**
  (omita-a). Devolve as linhas criadas com o `id` preenchido.
- **UPDATE** — atualiza as linhas; cada objeto de `data` **deve conter `id`** (a linha a alterar) mais
  os atributos a mudar.
- **DELETE** — remove as linhas; cada objeto de `data` identifica a linha por **`id`**.

> **Controle de acesso.** `_conf.accessControl` é uma lista de **papéis** por operação (`read`, `write`).
> Autoriza a **entidade inteira**, não a linha: basta um papel do usuário constar da lista para que ele
> escreva em qualquer registro da tabela. Não existe forma de expressar "só a própria linha" no
> `accessControl` — regra de dono só existe em processor `br`, que o `/e` não executa. Ver
> [forger/entity](../../forger/endpoints/entity.md).

## Respostas

| Ação | Sucesso | Corpo |
|---|---|---|
| `CREATE` | `201` | linhas criadas, com `id` gerado |
| `UPDATE` | `200` | vazio |
| `DELETE` | `200` | vazio |
| lote (array) | `200` | vazio |

Ação desconhecida → `400`. Falha de aplicação → `510`.

## Comportamento

A operação é aplicada **diretamente** à tabela da entity no banco do tenant, na requisição — **sem**
comando, **sem** evento e **sem** validação de transição de estado. Não há reconstituição de agregado:
é escrita de estado (linha), identificada pela chave primária `id`.

## Ler (R)

A leitura **não** é feita por `/e`. Para ler entidades (por chave, por critério, paginado), use
[persistence-q — consulta](../../persistence-q/endpoints/consulta.md).

## Erros

`400` (ação/corpo inválido; identificador ausente), `403` (tenant não autorizado), `510` (falha de
aplicação — ex.: violação de restrição, dado inconsistente; categoria **Aplicação de projeção**).
Catálogo: [../erros.md](../erros.md).
