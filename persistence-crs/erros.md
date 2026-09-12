# persistence-crs · catálogo de erros

> Envelope e categorias de erro. Guia: [README.md](README.md).

## Envelope

Em erro, o serviço devolve o status HTTP e um corpo com a mensagem. Erros relevantes também são
publicados de forma assíncrona para o subsistema de monitoração (sem PII).

## Códigos HTTP

| HTTP | Significado | Quando ocorre |
|---|---|---|
| `400` | Requisição inválida | Corpo malformado; comando/ação desconhecida; identificador inválido; **valueObject na forma errada** (ver [spec/model-format.md § data.valueObject](spec/model-format.md) — a mensagem diz a forma esperada). |
| `403` | Não autorizado | `tenant-id` não pertence ao usuário; credencial inválida; **usuário sem papel autorizado para o comando** (os papéis vêm de `command.<cmd>.roles` no modelo). |
| `510` | Falha de estado/processamento | Transição inválida; conflito de concorrência; **identificador de agregado malformado** (UUID inválido — ex.: enviar a PK `id` Long da projeção onde a URL `/a/{bc}/{type}/{id}` espera o `aggregateid`/UUID → "Invalid UUID string"); falha ao aplicar regra/coordenação; falha de projeção; demais exceções. |

## Categorias (para diagnóstico)

As falhas são classificadas em categorias que ajudam o agente a decidir a correção:

| Categoria | Causa | Correção |
|---|---|---|
| Validação | Cabeçalho/payload malformado; identificador inválido; forma do `valueObject` divergente da declarada no modelo. | Corrigir a requisição. |
| Autorização | Tenant fora do escopo do usuário; usuário sem o papel exigido pelo comando. | Usar credencial/tenant corretos; obter o papel declarado em `roles`. |
| Provisionamento de tenant | Tenant não provisionado / esquema indisponível. | Concluir o deploy (forger) antes de operar. |
| Busca de modelo | Agregado/comando ausente no modelo publicado. | Publicar/corrigir o **model** do tenant. |
| Transição de estado | Estado atual não permite o comando (`fromState`). | Enviar o comando adequado ao estado atual. |
| Concorrência | Conflito otimista entre comandos no mesmo agregado. | Reenviar sobre o estado atualizado. |
| Regra/coordenação | br-service indisponível ou rejeitou. | Verificar a função no br-service e os dados. |
| Infraestrutura | Falha de broker/banco/cache. | Retentar; se persistir, escalar. |
| Aplicação de projeção | Falha ao materializar a projeção (restrição, dado inconsistente). | Conferir a definição da entity e os dados. |
| Desconhecida | Não classificada. | Escalar com o contexto da requisição. |

## O comando respondeu `200` e a linha não apareceu

> **É o sintoma mais caro da plataforma, e o motivo é que o `200` está certo.**

**O `200` responde pela ESCRITA, não pela leitura.** O comando valida a transição e grava o evento —
é isso que ele confirma. A projeção é aplicada **depois, em outro processo, a partir de uma fila**.
Quando ela falha, quem chamou já foi embora, e não há para quem responder.

**Portanto: depois de qualquer transição nova, confira a projeção.** O `200` do comando não prova que a
leitura acompanhou. Vale sobretudo na primeira vez que um comando novo roda em um tenant — é aí que
modelo e projeção podem discordar sem ninguém saber.

### Se a linha não apareceu, procure o rastro

A falha **fica registrada**. Consulte o log do serviço na janela em que o comando rodou:

```
POST /v3/persistence/t/c/logs/service/crs/query/NAO-MATERIALIZOU/from/<início>/to/<fim>
X-Tenant-Id: <tenant>
```

> ⚠️ **`204` aqui significa "não achei esse termo", não "não houve falha".** O termo vai na própria
> rota e a busca é literal — errar a palavra devolve vazio, que é indistinguível de sucesso. Se vier
> `204`, confira o termo antes de concluir qualquer coisa.
>
> **Vigência:** o termo único `NAO-MATERIALIZOU` existe a partir da imagem que carregar
> `persistence-crs@9bc1b28`. Antes dela, a mesma falha saía sob **três** termos, conforme o caminho —
> `receiveMessage-EXC` (projeção do próprio tenant), `cross-BC` (projeção entre contextos) e
> `cross-coordination` (saga). Em imagem anterior, procure os três.

**Espere uma linha por tentativa.** A entrega é retentada **3 vezes** antes de a mensagem ser
estacionada; três registros do mesmo `eventId` são o esperado, não três falhas distintas.

### Como ler o que voltou

| O que a mensagem diz | O que aconteceu | Correção |
|---|---|---|
| `unknown attribute or component '<entity>.<atributo>'` | o modelo escreve um atributo que a entity não tem. Costuma ser o **carimbo do evento** (`whenAttribute`), esquecido quando se criam só as colunas dos atributos | criar a coluna na entity |
| `duplicate key value violates unique constraint` | a entity impõe unicidade que o modelo não declara mais — resíduo de um `identity.fields` anterior. **Mudar o modelo não derruba índice já existente** | `PUT` na entity com `unique: false`, ou reveja o `identity.fields` |
| `BR falhou statusCode=…` | a regra de negócio recusou ou está fora do ar | ver o br-service |
| `BR não retornou processedData.targetCommand` | a coordenação não produziu comando-alvo: **a saga parou aqui** | ver o retorno do processador |
| `payload sem …, descartando` | a mensagem chegou incompleta e foi descartada | reportar — é defeito de quem publicou |

### E se não houver rastro nenhum

Então **o evento não chegou até aqui**, e a causa está **antes** do motor: no que montou a requisição,
na publicação do modelo, ou no processador. Este catálogo não a cobre — comece pelo mais barato:
**mande o mesmo comando variando o VALOR do campo que sumiu**, antes de investigar transporte, cache
ou infraestrutura. Campo que chega vazio e campo que não chega podem ser indistinguíveis para quem
monta o payload, e o teste custa segundos.

## Nota
Um erro de **transição** ou de **concorrência** não é falha de rede: é resposta de domínio. O agente
deve tratá-los como sinais de fluxo (ajustar o comando ou reenviar), não como indisponibilidade.
