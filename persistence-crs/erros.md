# persistence-crs · catálogo de erros

> Envelope e categorias de erro. Guia: [README.md](README.md).

## Envelope

Em erro, o serviço devolve o status HTTP e um corpo com a mensagem. Erros relevantes também são
publicados de forma assíncrona para o subsistema de monitoração (sem PII).

## Códigos HTTP

| HTTP | Significado | Quando ocorre |
|---|---|---|
| `400` | Requisição inválida | Corpo malformado; comando/ação desconhecida; identificador inválido. |
| `403` | Não autorizado | `tenant-id` não pertence ao usuário; credencial inválida; **usuário sem nenhum dos papéis declarados em `command.<comando>.roles`** (ver nota abaixo). |
| `510` | Falha de estado/processamento | Transição inválida; conflito de concorrência; **identificador de agregado malformado** (UUID inválido — ex.: enviar a PK `id` Long da projeção onde a URL `/a/{bc}/{type}/{id}` espera o `aggregateid`/UUID → "Invalid UUID string"); falha ao aplicar regra/coordenação; falha de projeção; demais exceções. |

## Categorias (para diagnóstico)

As falhas são classificadas em categorias que ajudam o agente a decidir a correção:

| Categoria | Causa | Correção |
|---|---|---|
| Validação | Cabeçalho/payload malformado; identificador inválido. | Corrigir a requisição. |
| Autorização | Tenant fora do escopo do usuário. | Usar credencial/tenant corretos. |
| Provisionamento de tenant | Tenant não provisionado / esquema indisponível. | Concluir o deploy (forger) antes de operar. |
| Busca de modelo | Agregado/comando ausente no modelo publicado. | Publicar/corrigir o **model** do tenant. |
| Transição de estado | Estado atual não permite o comando (`fromState`). | Enviar o comando adequado ao estado atual. |
| Concorrência | Conflito otimista entre comandos no mesmo agregado. | Reenviar sobre o estado atualizado. |
| Regra/coordenação | br-service indisponível ou rejeitou. | Verificar a função no br-service e os dados. |
| Infraestrutura | Falha de broker/banco/cache. | Retentar; se persistir, escalar. |
| Aplicação de projeção | Falha ao materializar a projeção (restrição, dado inconsistente). | Conferir a definição da entity e os dados. |
| Desconhecida | Não classificada. | Escalar com o contexto da requisição. |

## Nota
Um erro de **transição** ou de **concorrência** não é falha de rede: é resposta de domínio. O agente
deve tratá-los como sinais de fluxo (ajustar o comando ou reenviar), não como indisponibilidade.

## Nota — `403` por **papel** (`roles`) e o que ele substituiu

Comando cujo usuário não tem nenhum dos papéis de `command.<comando>.roles` é recusado com **`403`** e a
mensagem `O usuário '<usuário>' não possui papel autorizado para executar o comando '<comando>'.`

A recusa acontece **antes** da regra de negócio — logo o comando recusado **não** aciona a regra, e as
mensagens dela (ex.: "referência não existe", "campo obrigatório ausente") **não** chegam a quem não tem
o papel.

> **Mudou em 2026-08-31.** Antes a verificação corria **depois** da regra e a recusa saía como **`510`**
> com mensagem interna. Quem tratava esse `510`, ou esperava a mensagem da regra nesse cenário, precisa
> ajustar. Detalhe: [README — autorização por papel](README.md#autorizacao-por-papel).

## Nota — `510` de **modelo do tenant ausente do cache**

Vale o mesmo que em [persistence-q/erros.md](../persistence-q/erros.md): acontece quando o **tenant é
válido** mas o **modelo dele não está no cache** (tenant novo sem modelo publicado, ou entrada expirada).
A mensagem **atual** nomeia a causa e a chave que faltou; a **anterior** — `Falha crítica. X-Tenant-Id não
reconhecido.` — ainda pode aparecer em ambiente que não recebeu a versão nova, e é enganosa: aponta para o
cabeçalho, mas a causa é o modelo. Nos dois casos a ação é a mesma: confirmar a publicação do modelo e o
dataschema em `RUNNING`, não mexer no cabeçalho.
