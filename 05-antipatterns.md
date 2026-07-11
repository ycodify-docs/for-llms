# Antipadrões — o que mais quebra (por serviço)

> Erros recorrentes que fazem um agente falhar, consolidados por serviço. Cada item: **sintoma →
> causa → correção**. Complementa os "pitfalls" de cada `README`. Pré-requisitos: [conceitos](02-conceitos.md).

## Contents
- Transversais
- forger
- persistence-crs
- es-n
- persistence-q
- br-service

---

## Transversais

- **Ler logo após escrever e achar vazio** → a projeção é **assíncrona** após o comando → tolerar a
  defasagem / reconsultar. Ver [persistence-q](persistence-q/README.md#ciclo-de-vida-de-uma-consulta).
- **Esquecer cabeçalhos** → comando/consulta sem `Authorization` **e** `X-Tenant-Id` → `403`/rejeição.
- **Achar que o `.model.json` cria filas/tabelas** → não cria: tabelas vêm de `entity`; filas, de
  `process`/es-n. Ver [arquitetura — de onde vêm as filas](01-arquitetura.md#origem-das-filas).

## forger

- **Criar recurso fora de ordem** → falta o "pai" → erro. Ordem: dbconn → database → (project,
  dataschema) → entity → model. Ver [fluxo de deploy](03-fluxo-de-deploy.md).
- **Publicar `model` antes das `entity`** → projeção sem tabela → leitura falha depois. Crie as entities primeiro.
- **Criar/alterar `entity` com dataschema em `RUNNING`** → o forger **rejeita** alterações de schema:
  transite `RUNNING → MODELING`, edite, volte a `RUNNING`. Ver [dataschema — gate](forger/endpoints/dataschema.md#atualizar).
- **Esquecer de ativar o dataschema (`RUNNING`) após o deploy** → comandos/consultas **não** são
  interpretados enquanto em `MODELING`. Ative antes de operar (passo 7 do [fluxo de deploy](03-fluxo-de-deploy.md)).
- **Nome fora do formato/limite** → `400`. Limites: project ≤16, dataschema ≤16, entity/atributo/assoc
  ≤24 (`^[a-z]+$`). Ver [forger/README — regras de nome](forger/README.md#regras-de-nome-dos-recursos).
- **Declarar/enviar atributo reservado** (`id`, `logversion`, `logrole`, `loguser`) → rejeição.
- **`PUT` misturando escopos** (`_conf` + atributos + associações na mesma chamada) → use **um escopo por requisição**.
- **`logversion` desatualizado no `PUT`** → `409` → reler e reenviar com a versão atual.
- **Remover recurso com dependentes** → bloqueado → remover os dependentes antes.
- **Comentário sem `_` no topo do `.model.json`** → vira "root key" e quebra (`400`); use `_comment`.
  Chaves `_`-prefixadas são removidas (qualquer nível) na publicação do modelo.
- **Dropar `_conf` da definição de entity** por confundir com a convenção do modelo → `_conf` é
  **obrigatório/semântico** na entity (outro contrato); a regra "_ = removido" é só do `.model.json`.

## persistence-crs

- **Enviar `status` = estado pretendido (`endState`)** → errado: `status` = **estado atual** do
  agregado; divergência → `510`. Exceção: comando de **criação** não envia `status`.
- **Enviar `id` na criação** → o `id` é **gerado** (UUID); omita.
- **Usar a PK `id` (Long) da projeção como id do agregado** → nos endpoints persistence-crs
  (`/a/{bc}/{type}/{id}`: comando de transição, leitura de estado, `/history`) o `{id}` é o **UUID** do
  agregado (o `aggregateid` da projeção), **não** a PK Long da linha. Enviar o Long → `510`
  "Invalid UUID string". Lembre: `projecao.aggregateid == aggregate.id`.
- **Operar com o dataschema em `MODELING`** → o modelo **não é interpretado** nesse estado e o comando
  falha; o dataschema precisa estar em `RUNNING`. Ver [dataschema — gate](forger/endpoints/dataschema.md#atualizar).
- **Enviar campo de timestamp do evento** (`whenAttribute`, ex. `criadaem`) → é auto-valorizado; não envie.
- **Comando inválido para o estado atual** (`fromState`) → `510` de transição (não é falha de rede):
  envie antes o comando que leva ao estado de origem exigido.
- **Embrulhar o comando** em wrapper extra → envie `{ "<comando>": { ...dados } }`.
- **Assumir entrega sem idempotência** → projeção é ao-menos-uma-vez na fila / exatamente-uma-vez na
  aplicação; processadores devem ser idempotentes.

## es-n

- **Ordenação estrita "por garantia"** → reduz paralelismo; use **por-agregado** (padrão) salvo
  necessidade real de ordem global.
- **Esperar fila dedicada por par de contextos** → os canais cross-contexto/coordenação são
  **universais e fixos**; o destino vai no **conteúdo** da mensagem. Ver [es-n](es-n/README.md).
- **Declarar `triggerProjection`/`triggerCoordination` esperando que criem filas** → não criam; só roteiam.

## persistence-q

- **Inventar predicado** fora do vocabulário do tenant → `510`. Use o vocabulário do modelo/projeção.
- **Rótulos repetidos no modo array** → resultados ficam indistinguíveis; use rótulos únicos.
- **Tratar `204` como erro** → `204` = "nenhum resultado".
- **Ignorar itens vazios** → num `200`, critérios sem resultado vêm como `[]` na mesma posição.

## br-service

- **Chamar o br-service diretamente** → ele é acionado **pelo persistence-crs** conforme o modelo.
- **Rota não-canônica** → colisão entre organizações; use `<org>/<project>/<bc>/<aggregate>/…`. Ver
  [br — forma canônica](br-service/README.md#forma-canônica-da-rota-obrigatória).
- **Devolver envelope em vez do objeto direto** → o sucesso (`200`) é o **objeto do processador** sem envelope.
- **Efeito colateral de escrita no caminho crítico** → processadores devem ser idempotentes; leitura de
  enriquecimento é aceitável, escrita externa não.
