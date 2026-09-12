# persistence-crs — guia do serviço

> **Papel:** executa os **comandos** definidos para um agregado num bounded context. É o **lado de
> escrita** do CQRS-ES: valida a transição de estado, eventualmente aciona regra/coordenação e **grava
> o evento**.
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md). Depende do
> **model** publicado pelo forger (CP-1).

## Contents
- Pré-requisitos do chamador
- Índice de endpoints
- Ciclo de vida de um comando
- Estados, transições e concorrência
- Pontos de coordenação
- Pitfalls

---

## Pré-requisitos do chamador

- **Cabeçalhos obrigatórios:** `Authorization` e o **cabeçalho de tenant** `X-Tenant-Id`.
- **Autorização de tenant:** o `tenant-id` deve pertencer ao usuário; caso contrário → `403`.
- **Modelo publicado:** o agregado/comando precisa existir no **model** publicado para o tenant (CP-1).
  Comando/agregado desconhecido → erro de busca de modelo.
- **Dataschema em `RUNNING`:** o serviço **só interpreta/executa** o modelo se o dataschema do tenant
  estiver em `RUNNING`. Em `MODELING` (esquema em edição) a interpretação **não ocorre** e o comando
  falha. Ver [forger/dataschema — gate de status](../forger/endpoints/dataschema.md#atualizar).

## Índice de endpoints

> Base do serviço: `/a` (agregados). Apenas endpoints autenticados são documentados. Endpoints internos
> sem autenticação e endpoints de observabilidade/administração **não** constam aqui.

| Operação | Método · Path | Documento |
|---|---|---|
| Executar comando | `POST /a/{boundedContext}/{aggregateType}` | [endpoints/comando.md](endpoints/comando.md) |
| Ler estado do agregado | `GET /a/{boundedContext}/{aggregateType}/{id}` | [endpoints/agregado-leitura.md](endpoints/agregado-leitura.md) |
| Ler histórico de eventos | `GET /a/{boundedContext}/{aggregateType}/{id}/history` | [endpoints/agregado-leitura.md](endpoints/agregado-leitura.md) |
| Escrever/remover linha de entidade | `POST /e` | [endpoints/entidade.md](endpoints/entidade.md) |
| Remover linhas por predicado | `POST /e/by` | [endpoints/remocao-por-predicado.md](endpoints/remocao-por-predicado.md) |
| Escrever projeção (mesma rota `/e`) | `POST /e` | [endpoints/projecao.md](endpoints/projecao.md) |
| Consultar logs do serviço | `GET /logs/service/{service}/query/…` | [endpoints/logs.md](endpoints/logs.md) |

Gramática do modelo de domínio: **[spec/model-format.md](spec/model-format.md)** (estrutura do
`.model.json`: agregado, comando, evento, tipos). Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md).

## Ciclo de vida de um comando

`POST /a/{boundedContext}/{aggregateType}`:

1. **Auth + tenant** — valida `Authorization` e o `tenant-id`; carrega o **modelo** do tenant (CP-1).
2. **Decomposição** — identifica o agregado, o **comando** e seus dados a partir do corpo.
3. **Carga do estado atual** — reconstitui o estado do agregado a partir dos seus eventos.
4. **Validação de transição** — o estado atual precisa pertencer ao conjunto de **estados de origem**
   (`fromState`) do comando; senão → erro de transição.
5. **Regra/coordenação (se declarada)** — se o modelo do comando declara uma rota de **regra de
   negócio** ou de **coordenação**, o serviço chama o **br-service** para validar/enriquecer/coordenar
   **antes** de concluir (CP-6).
6. **Validação do estado de destino** — após a regra, o estado resultante deve corresponder ao
   **estado de destino** (`endState`) previsto.
7. **Gravação do evento** — grava o evento no banco de escrita (universal, isolado por `tenant-id`).
   A gravação, na mesma transação, **notifica o es-n** (CP-3).
8. **Resposta** — `200` com `{ "id": "<id do agregado>", "status": "<estado>" }`.

A atualização da **projeção** acontece **depois**, de forma assíncrona, pelo fluxo es-n → projeção
(CP-4/CP-5). A resposta do comando **não** garante que a projeção já reflete a mudança.

## Estados, transições e concorrência

- Um comando só é aceito se o agregado está num **estado de origem** válido.
- No modelo, cada **estado de origem** de um comando deve ser **estado de destino** de algum outro
  comando (ou o estado inicial) — isso garante uma máquina de estados conexa, sem transições órfãs.
- **Regra do `status` (concorrência otimista):**
  - **Todo** comando carrega o campo `status` **valorado**, com **uma exceção**: o **comando inicial**
    (criação), que **não** envia `status` (o agregado ainda não existe).
  - O valor de `status` é **sempre o estado atual do agregado no banco de escrita** — **nunca** o estado
    pretendido ao fim da execução (`endState`).
  - Se o `status` enviado divergir do estado real (outro comando avançou antes), a operação é rejeitada
    (`510`) e deve ser **reenviada** sobre o estado atualizado.

## Regras de payload (checklist)

- [ ] **Sem wrapper extra** — enviar o comando pelo seu nome: `{ "<comando>": { ...dados } }`.
- [ ] **Não enviar identidade na criação** — o `id` do agregado é **gerado** pela plataforma; omita-o
      no comando de criação.
- [ ] **Não enviar campos de timestamp de evento** (`whenAttribute`) — são **valorizados
      automaticamente** quando o evento é gravado. Convenção de nome: particípio passado do verbo do
      comando + sufixo "em" (ex.: comando `criar` → evento `criada` → campo `criadaem`).
- [ ] **`status` (concorrência):** enviar **sempre** o `status` = **estado atual** do agregado (do
      banco de escrita), **nunca** o `endState` pretendido. **Única exceção:** o comando de **criação**
      não envia `status`. Divergência com o estado real → `510`.
- [ ] **Value objects:** dados aninhados seguem a forma declarada no modelo (objeto único ou lista),
      conforme o agregado.

## Garantia de entrega e projeção

- A gravação do evento é **transacional**; a notificação ao es-n ocorre na **mesma transação**.
- A entrega na fila é **ao-menos-uma-vez**; a aplicação na projeção é **exatamente-uma-vez** (há
  deduplicação por evento). Logo, processadores de projeção devem ser **idempotentes**.
- O **modelo** do tenant é lido de um cache (ver seção abaixo); alterar o modelo exige **republicá-lo**
  (forger) **+ invalidar o cache** — a mudança **não** é refletida de imediato.

## Carga da spec do tenant (cache por-instância)

A cada requisição (com `X-Tenant-Id`), o serviço resolve a spec do tenant (entities + `.model.json`) assim:

1. **Consulta o Forger** o status do dataschema do tenant — se **não** estiver `RUNNING` → **exceção** (o modelo não é interpretado).
2. Usa a spec da **memória local** da instância, se presente → segue.
3. Senão, lê do **serviço de cache** (`../cache`) e a carrega na memória local.
4. Se **não** houver no cache → **exceção `510`**, indicando que o modelo do tenant não está no cache e
   precisa ser **republicado**.

> **Não há recuperação automática a partir do Forger (não é self-heal).** O modelo entra no cache
> **apenas** quando é publicado ([forger/model](../forger/endpoints/model.md)) — a publicação é a
> **única** porta de entrada. Um miss **não** é um caminho lento que se resolve sozinho: é **parada**.
> Como a entrada **não expira** por conta própria, um miss significa que o modelo nunca foi publicado
> ou que a entrada foi removida — e a saída é **republicar**, não reiniciar nem esperar.

### Depois de alterar o modelo

**Quem publica e quem remove os modelos do cache é o forger, e o procedimento é a fatia dele:**
[forger/dataschema — o que a transição faz com o modelo no cache](../forger/endpoints/dataschema.md#o-que-a-transição-faz-com-o-modelo-no-cache).
Lá está o roteiro completo e o pré-requisito para voltar a `RUNNING`. **Não repito aqui:** mecanismo com
duas casas diverge, e quando diverge as duas parecem autoridade.

**O que é meu, e é só isto: o que este serviço faz quando o modelo não está no cache.**

- **Falha explícito, com `510`**, e a mensagem manda **republicar o modelo**. Não há degradação
  silenciosa: o serviço nunca responde `200` com menos dado porque o modelo faltava.
- Vale para **comando e consulta**, e vale **enquanto o dataschema estiver em edição** (`MODELING`) — é
  nessa janela que o modelo não está publicado. **O que morde não é editar: é deixar a edição aberta.**

> ⚠️ **Não invalide cache à mão para propagar uma alteração.** A publicação e a remoção são da
> plataforma. **Isto corrige o que este documento dizia até 2026-09-10**, quando mandava "invalidar as
> DUAS chaves do modelo" antes de republicar — trabalho desnecessário. Se você seguia aquele texto, pode
> parar.

**Sobre cópia em memória do serviço.** Até `2026-09-10` cada instância guardava o modelo em memória por
até uma hora, e a alteração podia "pegar" numa instância e não noutra. **Na instância de teste
(`tinterpreter`), essas cópias estão desligadas desde a imagem `amd64-260910`**: o modelo é relido do
serviço de cache a **cada requisição**, e republicar passa a valer na requisição seguinte. Onde elas
continuarem ligadas, a defasagem de até uma hora ainda vale e o descarte da cópia é operação de
plataforma. **Reiniciar o serviço nunca foi remédio** — ele repuxa o que estiver no cache.

> ⚠️ **O `200` do comando não prova que a projeção acompanhou.** Ele responde pela escrita; a projeção
> é aplicada depois, a partir de uma fila. Se a linha não aparecer, a falha **está registrada** — a
> receita para achá-la está em
> [erros.md § O comando respondeu 200 e a linha não apareceu](erros.md#o-comando-respondeu-200-e-a-linha-não-apareceu).

## Pontos de coordenação

- **CP-3** — gravar o evento **notifica o es-n** (mesma transação).
- **CP-6** — o serviço **chama o br-service** quando o modelo do comando o exige (regra/coordenação).
- **CP-5/CP-7** — consome filas (projeção, coordenação) para aplicar projeções e fechar sagas.

Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Enviar **sempre** `Authorization` **e** o cabeçalho de tenant.
- [ ] Garantir que o **model** do tenant está publicado e contém o agregado/comando (senão, erro de modelo).
- [ ] Respeitar a transição: comando inválido para o estado atual → erro de transição (não é falha de rede).
- [ ] Tratar **conflito de concorrência** reenviando o comando sobre o estado mais novo.
- [ ] Não assumir leitura imediata: após o comando, a **projeção** é atualizada de forma assíncrona —
      para ler, use [persistence-q](../persistence-q/README.md), tolerando um curto atraso.
