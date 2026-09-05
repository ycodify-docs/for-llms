# persistence-q — guia do serviço

> **Papel:** **lado de leitura** do CQRS. Consulta o estado dos agregados pela ótica de suas
> **projeções** materializadas. Não executa comandos nem reprocessa eventos.
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md). Lê as projeções
> criadas pelo forger (CP-2) e mantidas pelo fluxo es-n → projeção (CP-5).

## Contents
- Pré-requisitos do chamador
- Índice de endpoints
- Ciclo de vida de uma consulta
- Depois de um comando, espere antes de consultar
- Linguagem de consulta (resumo)
- Pontos de coordenação
- Pitfalls

---

## Pré-requisitos do chamador

- **Cabeçalhos obrigatórios:** `Authorization` e o **cabeçalho de tenant** `X-Tenant-Id`.
- **Autorização de tenant:** o `tenant-id` deve pertencer ao usuário; senão → `403`.
- **Vocabulário do tenant:** os identificadores de consulta e predicados válidos são definidos pelo
  **model**/projeções provisionados para o tenant (não inventar predicados).
- **Dataschema em `RUNNING`:** a consulta **só é interpretada/executada** se o dataschema do tenant
  estiver em `RUNNING`. Em `MODELING` (esquema em edição) a interpretação **não ocorre**. Ver
  [forger/dataschema — gate de status](../forger/endpoints/dataschema.md#atualizar).
- **Spec carregada por-instância (cache):** a cada consulta o serviço resolve a spec do tenant igual ao
  write-side — dataschema `RUNNING` via Forger → memória local → **serviço de cache** (`../cache`) →
  **se falta no cache, a consulta falha** (`510`, "republique o modelo"). **Não há recuperação
  automática a partir do Forger:** o modelo entra no cache **só** quando é publicado, e a entrada
  **não expira** sozinha. Alterar o modelo só propaga após **invalidar o cache** (as duas chaves do
  modelo — read-model + write-model, valores internos) + republicar; **restart sozinho não resolve**.
  Detalhe: [persistence-crs/README §Carga da spec do tenant](../persistence-crs/README.md).
- **Dois identificadores na projeção:** cada linha tem `id` (**Long**, PK da linha) e `aggregateid`
  (**UUID** de 36 chars, o id do agregado; `projecao.aggregateid == aggregate.id`). Filtra-se por
  `aggregateid` para achar a projeção de **um** agregado específico. Para operar o agregado depois
  (comando/`/history` na [persistence-crs](../persistence-crs/README.md)), usa-se o **UUID**
  (`aggregateid`), **nunca** a PK `id` (Long) — o Long num endpoint de agregado → `510`
  "Invalid UUID string".

## Índice de endpoints

> Apenas o endpoint autenticado é documentado. Endpoints internos sem autenticação e endpoints de
> observabilidade/administração **não** constam aqui.

| Operação | Método · Path | Documento |
|---|---|---|
| Consultar projeções | `POST /` | [endpoints/consulta.md](endpoints/consulta.md) |
| Controles da DSL (operadores, paging, sorting, cache, associações) | — | [query-controls.md](query-controls.md) |

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md).

## Ciclo de vida de uma consulta

`POST /`:

1. **Auth + tenant** — valida `Authorization` e o `tenant-id`.
2. **Detecção de modo** — pelo primeiro caractere do corpo: `[` → **array** (múltiplos critérios);
   `{` → **object** (critério único).
3. **Leitura das projeções** — para cada critério, lê a projeção correspondente no banco de leitura,
   aplicando os predicados informados.
4. **Resposta** — sempre um **array no topo**, na **mesma ordem** dos critérios; `200` se houver algum
   resultado, `204` se todos vierem vazios.

A consulta é **idempotente** e **não** modifica estado.

> **Consistência eventual:** a projeção é atualizada de forma assíncrona após cada comando. Logo após
> um comando bem-sucedido, a consulta pode ainda **não** refletir a mudança por um curto intervalo.

### Depois de um comando, espere antes de consultar

**Regra:** todo polling que consulta este serviço logo após um comando deve começar por um **atraso
inicial (em milissegundos)** e depois **reconsultar**, com número de tentativas limitado. **Não** dispare
a primeira consulta junto com a resposta do comando.

**Por que a regra existe:** o mecanismo que persiste a **projeção** é **desvinculado** do mecanismo que
persiste o **agregado**. São dois caminhos independentes — o comando grava o evento e responde; a
projeção é aplicada **depois**, por outro caminho (o despacho assíncrono do evento). Logo:

| O que o `200` do comando afirma | O que ele **não** afirma |
|---|---|
| o evento foi gravado e a transição era válida | que a linha da projeção já existe ou já está atualizada |

Consequências práticas:

- **Consultar imediatamente devolve o estado anterior** — e isso **não** é erro, nem do comando nem da
  consulta. Tratar esse resultado como verdade leva à conclusão errada de que o comando falhou.
- **Não há número universal para o atraso.** Ele varia com a carga do despacho e com o tamanho do
  modelo. Por isso a regra é *atraso inicial + reconsulta limitada*, nunca uma consulta única.
- **Reconsultar é seguro:** a consulta é idempotente e não modifica estado.
- **Uma falha na projeção não aparece na resposta do comando.** Se a linha nunca chegar, a causa está no
  caminho assíncrono, não na resposta que o cliente recebeu — investigar só a resposta do comando não
  encontra nada.

## Linguagem de consulta (resumo)

- Cada **critério** é um objeto com **um** identificador de consulta (rótulo escolhido pelo cliente)
  mapeando para um conjunto de **predicados** (pares chave-valor, combinados por "E").
- O rótulo aparece **literalmente na resposta**, agrupando os registros encontrados.
- O **vocabulário** de predicados é definido pelo modelo do tenant; nome desconhecido → erro de
  tradução de critério.

Detalhe e exemplos: [endpoints/consulta.md](endpoints/consulta.md).

## Pontos de coordenação
- **CP-2** — consulta as projeções (tabelas) criadas pelo forger.
- **CP-5** — lê o que o fluxo es-n → projeção mantém atualizado.
Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Enviar **sempre** `Authorization` **e** o cabeçalho de tenant.
- [ ] Usar **rótulos únicos** por critério no modo array (rótulos repetidos misturam resultados).
- [ ] Não inventar predicados — usar o vocabulário provisionado para o tenant.
- [ ] Tolerar **atraso de propagação**: logo após um comando, a projeção pode ainda não refletir a
      mudança (atualização assíncrona).
- [ ] Tratar `204` como "nenhum resultado", não como erro.
