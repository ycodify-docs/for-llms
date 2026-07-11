# Conceitos e vocabulário

> Ontologia mínima da plataforma e o **vocabulário público** usado em toda a documentação.
> Este arquivo é a **fonte única** dos termos: todo outro documento usa exatamente estes nomes.

## Contents
- Modelo mental (DDD + CQRS-ES)
- Glossário de recursos e domínio
- Identidade e isolamento (tenant)
- Vocabulário de infraestrutura (termos abstratos)

---

## Modelo mental (DDD + CQRS-ES)

A plataforma materializa um sistema a partir de **modelos declarativos**, não de código imperativo.
O cliente descreve seu domínio (contextos, agregados, comandos, eventos, projeções) e a plataforma
interpreta esses modelos em tempo de execução.

- **CQRS** — a escrita (executar comandos) e a leitura (consultar) são caminhos separados, com
  modelos de dados distintos.
- **Event Sourcing (ES)** — o estado de um agregado é a soma dos seus eventos; cada mudança de estado
  é registrada como um evento imutável no **armazém de eventos** (banco de escrita).
- **Projeção** — uma visão materializada do estado atual de um agregado, mantida no **banco de leitura**
  (relacional, SQL), atualizada de forma assíncrona após cada evento.

---

## Glossário de recursos e domínio

| Termo | Definição |
|---|---|
| **dbconn** | Metadados de conexão a um servidor de banco relacional. Primeiro recurso a existir. |
| **database** | Banco de dados (de leitura) vinculado a um `dbconn`. |
| **project** | Um **bounded context** (contexto delimitado, no sentido DDD). Unidade de organização do domínio. |
| **dataschema** | Esquema de dados vinculado a um `project` e a um `database`. Âncora do isolamento por tenant. |
| **entity** | Definição de uma **projeção** a ser materializada como tabela no banco de leitura. |
| **model** (modelo de domínio) | Documento declarativo (`.model.json`) que descreve agregados, comandos, eventos e suas transições para um bounded context. |
| **process** | Modelo de processo (orquestração) e de decisão, derivado de um diagrama, publicado para execução. |
| **agregado** (aggregate) | Unidade de consistência do domínio (DDD): agrupa estado + invariantes; alvo de comandos. |
| **comando** (command) | Intenção de mudança sobre um agregado. Pode ser aceito ou rejeitado conforme o estado atual. |
| **evento** (event) | Fato consumado: o registro imutável de uma mudança de estado de um agregado. |
| **projeção** (projection) | Visão de leitura do estado atual de um agregado, materializada como tabela no read-model. Toda linha tem **dois** identificadores: `id` (**Long**, PK da linha) e `aggregateid` (**UUID**, o id do agregado — `projecao.aggregateid == aggregate.id`). Todo agregado tem ao menos uma (comumente uma). |
| **transição de estado** | Um comando só é válido se o estado atual do agregado pertence ao seu conjunto de estados de origem; ao concluir, o agregado fica num estado de destino. |
| **regra de negócio** | Validação/enriquecimento aplicado aos dados de um comando antes da gravação do evento. |
| **coordenação** | Lógica que articula múltiplos agregados/contextos, comumente no contexto de uma **saga**. |
| **saga** | Fluxo de coordenação entre agregados/contextos, dirigido por eventos. |
| **value object** | Dado aninhado sem identidade própria, parte de um agregado. Na projeção vira coluna `Json`: `single` = objeto JSON (1:1), `multiple` = array de objetos JSON (1:N). |
| **`domainBus`** | Bloco do evento (no `.model.json`) que carrega o roteamento de despacho: `orderingMode`, `triggerProjection`, `triggerCoordination`. |
| **`whenAttribute`** | Atributo de timestamp do evento, valorizado automaticamente na gravação. Nome = particípio do comando + sufixo `em` (ex.: `criadaem`). |
| **`roles`** | Lista de papéis autorizados a executar um comando. |
| **identity** | Identificação do agregado: `strategy` = tipo do `id` (UUID, gerado pela plataforma); `fields` = vetor de nomes de atributos cuja **combinação de valores é única** (constraint de unicidade composta). |
| **concurrency** | Estratégia de concorrência do agregado (otimista), via campo de versão. |

> A gramática completa do `.model.json` (agregado, comando, evento, tipos) está em
> [persistence-crs/spec/model-format.md](persistence-crs/spec/model-format.md).

---

## Convenção padrão de topologia (project ↔ bounded context ↔ esquema)

Por **padrão** (recomendado, **não** obrigatório):

- um **project** corresponde a **um bounded context** (1:1);
- um **database** hospeda **um dataschema por bounded context** — ou seja, no banco de dados relacional
  há **um banco** e, dentro dele, **cada esquema representa um bounded context**.

```
database (1)
 ├── dataschema/esquema  →  bounded context A   (= project A)
 ├── dataschema/esquema  →  bounded context B   (= project B)
 └── …
```

Essa é a configuração padrão adotada. **Não é uma imposição:** outras topologias são possíveis (ex.:
mais de um esquema por contexto, ou organização distinta), conforme a necessidade do sistema.

## Do agregado à projeção (derivação e implantação)

A **projeção** de um agregado é uma **entity** materializada como **tabela no banco de leitura**, e suas
colunas **derivam dos atributos do agregado** — **tanto** os escalares (`data.attribute`) **quanto** os
value objects (`valueObject.single` → coluna `Json` objeto; `valueObject.multiple` → coluna `Json`
array). A cadeia que liga o agregado à sua projeção e a deixa implantada:

```
agregado  (vive num bounded context)
   │  bounded context = project           (convenção padrão 1:1)
   │  nome do bounded context = nome do dataschema  (no .model.json: schema.forReadModel.name)
   ▼
entity (projeção do agregado)             ← colunas derivam de data.attribute do agregado
   │  implantada via forger
   ▼
dataschema (= bounded context) ── vinculado a ──▶ project  e  database (banco de leitura)
```

Em palavras:

1. cada **agregado** pertence a um **bounded context**; por padrão, esse bounded context **é** um
   **project** (1:1);
2. o **nome do bounded context** equivale ao **nome do dataschema** de leitura
   (`schema.forReadModel.name` no `.model.json`);
3. a **entity** que é a **projeção** desse agregado é criada **via forger** **sob esse dataschema** —
   e o dataschema está vinculado ao **project** e ao **database** (banco de leitura);
4. logo, as **colunas** da projeção **derivam dos atributos** do agregado: `data.attribute` (colunas
   escalares) **e** `valueObject.single`/`valueObject.multiple` (colunas `Json` — objeto e array).

> Consequência prática: a projeção (entity) deve existir **antes** do primeiro comando contra o agregado,
> e seus atributos devem **espelhar** os atributos do agregado. Implantação: [forger/entity](forger/endpoints/entity.md).
> Tipos dos atributos: [model-format — tipos](persistence-crs/spec/model-format.md#atributos-e-tipos).

## Identidade e isolamento (tenant)

- **tenant-id** — chave que **identifica e isola** o processamento de comandos, eventos e projeções de
  um cliente. O isolamento ocorre em dois níveis: no serviço de comandos e no banco de dados (tanto de
  leitura quanto de escrita).
- **Banco de escrita universal** — o armazém de eventos é **comum a todos os clientes e a todos os
  agregados**; o isolamento lógico é garantido pelo `tenant-id`, não por bancos físicos separados.
- **Modelo de escrita vs modelo de leitura** — no modelo de domínio (`.model.json`):
  - `schema.forWriteModel.name` (armazém de eventos) é um **valor fixo definido pela plataforma**,
    **igual em todo agregado** (universal);
  - `schema.forReadModel.name` (projeção) é o **nome do dataschema** da projeção do agregado — por
    padrão, igual ao nome do **bounded context**, logo **varia por contexto**.
  - Ou seja: todos os agregados compartilham o mesmo armazém de eventos, mas cada contexto tem suas
    próprias projeções.
- **Header de tenant** — todas as requisições de comando, leitura e consulta exigem o cabeçalho que
  carrega o `tenant-id`. Sem ele, a requisição é rejeitada.

---

## Vocabulário de infraestrutura (termos abstratos)

Esta documentação descreve **contratos e comportamento**, não a implementação. Os componentes de
infraestrutura são referidos por papéis abstratos:

| Papel abstrato | O que faz |
|---|---|
| **cache distribuído** | Guarda modelos publicados (de domínio e de processo) para leitura rápida pelos serviços. |
| **broker de mensagens** | Transporta eventos despachados entre serviços, em filas, de forma assíncrona. |
| **banco de dados relacional (SQL)** | Banco de leitura (projeções) e banco de escrita (eventos). |
| **registro de serviços** | Permite que um serviço localize outro dinamicamente. |
| **fila de projeção** | Fila que conduz eventos para atualização da projeção no banco de leitura. |
| **fila de coordenação** | Fila que conduz eventos para coordenação entre agregados/contextos. |

> Esta documentação **não revela** produtos, versões de stack, nomes de filas/exchanges, nomes de
> classes, hosts, cabeçalhos internos ou variáveis de ambiente. Quando um detalhe desses seria
> necessário, ele é substituído pelo papel abstrato correspondente acima.
