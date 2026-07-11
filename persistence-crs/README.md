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
| Escrever projeção (serviço) | `POST /e` | [endpoints/projecao.md](endpoints/projecao.md) |

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
3. Senão, recupera do **serviço de cache** (`../cache`; **não** Redis direto) e a carrega na memória local.
4. Se **não** houver no cache, **recupera do Forger e repõe no cache** (self-heal), então carrega na memória local.
5. Se nem o Forger prover → **exceção**.

A spec fica **stale na memória local** e **não** refresca sozinha. Após alterar entity/model, o refresh exige
**invalidar as DUAS chaves de cache do modelo** — a do **read-model** **e** a do **write-model** (valores
internos, resolvidos pela plataforma/ops) — **além de** republicar o modelo + dataschema `RUNNING`.
**Reiniciar o serviço sozinho não resolve** (re-puxa a stale do `../cache`); **remover o cache sozinho** também
não (a instância viva mantém a stale na memória). O mesmo vale para o `persistence-q`.

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
