# persistence-crs · endpoints · comando

> Executa um **comando** sobre um agregado. Guia: [../README.md](../README.md).

## Requisição

`POST /a/{boundedContext}/{aggregateType}`

| Item | Valor |
|---|---|
| Cabeçalhos | `Authorization`, `X-Tenant-Id` (cabeçalho de tenant), `Content-Type: application/json`, `Content-Length` — o serviço o exige; cliente HTTP que use `Transfer-Encoding: chunked` sem `Content-Length` não casa a rota |
| `{boundedContext}` | contexto delimitado do agregado |
| `{aggregateType}` | tipo do agregado |

Corpo: o **comando** identificado pelo seu nome, com os dados:

```json
{ "<nomeDoComando>": { "<campo>": "<valor>", "...": "..." } }
```

O nome do comando, seus campos e as transições válidas são definidos no **model** publicado para o
tenant (ver [forger/model](../../forger/endpoints/model.md) e [examples/](../../examples/README.md)).

> **`id` (obrigatório, exceto na criação):** todo comando que **não** é de criação envia também o `id`
> (UUID) do agregado alvo — é ele que diz sobre qual agregado o comando age. O comando de **criação**
> (`fromState: []`) **não** o envia: o `id` é gerado pela plataforma e volta na resposta. Comando de
> transição sem `id` **não** é executado.

> **Campos de `valueObject` — duas formas, e só essas duas:** `single` recebe **um objeto**, `multiple`
> recebe **um array de objetos**. Vale para as **duas** maneiras de declarar o VO no modelo; a forma do
> valor é a mesma nas duas. **Escalar nunca** — nem solto, nem dentro de array: `"diassemana": ["terca"]`
> é recusado com `400`; a forma válida é `"diassemana": [{"<campo>": "terca"}]`. O que existe **dentro**
> do objeto é do seu domínio (um campo ou dez, com aninhamento). Num `multiple`, **um único item fora da
> forma reprova o comando inteiro**. Ver
> [spec/model-format.md § data.valueObject](../spec/model-format.md).

> **Campo `status` (obrigatório, exceto na criação):** todo comando envia `status` = **estado atual**
> do agregado (lido do banco de escrita), **nunca** o estado pretendido após o comando (`endState`). O
> **comando de criação** é a **única exceção** — não envia `status`. Divergência com o estado real →
> `510` (alguém avançou o agregado antes; reenvie sobre o estado atualizado). Ver
> [README — regra do status](../README.md#estados-transições-e-concorrência).

## Resposta

`200`:

```json
{ "id": "<id do agregado>", "status": "<estado resultante>" }
```

> Só `id` e `status` são devolvidos — **não** os dados do comando. A projeção é atualizada de forma
> assíncrona depois (ver [arquitetura](../../01-arquitetura.md)).

## Comportamento

A ordem real, que importa para entender o que já rodou quando um erro chega:

1. **autenticação e tenant** — o chamador tem de pertencer ao tenant do cabeçalho;
2. **carga do estado** do agregado;
3. **autorização por papel** (`command.<cmd>.roles`) — **antes** da regra de negócio. Quem não tem o
   papel recebe `403` **sem** o processor do br-service ter sido executado;
4. **regra/coordenação no br-service**, se o modelo declarar;
5. **validação da transição** (`fromState` contra o `status` enviado);
6. **gravação do evento**, que notifica o es-n;
7. resposta.

> A ordem entre 3 e 4 mudou em 2026-09-05. Antes o br-service rodava primeiro e a checagem de papel vinha
> depois — quem não tinha o papel fazia a regra executar inteira e recebia as mensagens dela de volta.
> Se você depende de ver a mensagem do processor para diagnosticar permissão, esse caminho não existe
> mais: sem papel, o `403` vem antes.

> **Não existe validação do estado de destino (`endState`).** Versões anteriores desta página descreviam
> um passo que compararia o estado resultante com o `endState` do modelo; ele não existe no serviço. O
> que é verificado é a **origem** (`fromState`), no passo 5.

## Erros

| Código | Quando |
|---|---|
| `400` | `valueObject` na forma errada (a causa mais comum) |
| `403` | tenant não autorizado, **ou** usuário sem o papel exigido pelo comando |
| `510` | falha de transição (`fromState` não casa o `status` enviado) **e o resto**: corpo malformado, agregado ou comando desconhecido, `id` ausente num comando de transição |

⚠️ **`510` não significa "erro do servidor" aqui.** Boa parte dos erros de payload cai nele — corpo
malformado, nome de comando inexistente, `id` faltando. Antes de suspeitar de indisponibilidade, leia a
mensagem: ela costuma dizer exatamente qual campo está errado.

Catálogo: [../erros.md](../erros.md).
