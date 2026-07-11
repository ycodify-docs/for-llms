# Fluxo de deploy — implantar um sistema do zero

> A sequência **obrigatória** para tornar um sistema operante, do nada até aceitar comandos e consultas.
> Todo este fluxo é conduzido pelo **forger**. Detalhe de cada endpoint: [forger](forger/README.md).
> Pré-requisitos: [conceitos](02-conceitos.md), [arquitetura](01-arquitetura.md).

> **Convenção padrão:** um **project** = um **bounded context**, e um **database** recebe **um
> dataschema por bounded context** (um esquema por contexto, no mesmo banco relacional). É o padrão
> recomendado, não obrigatório. Ver [conceitos — topologia padrão](02-conceitos.md#convenção-padrão-de-topologia-project--bounded-context--esquema).

## Por que existe uma ordem

Cada recurso **depende** do anterior. As dependências são impostas pela plataforma: não é possível criar
um recurso cujo "pai" ainda não exista, nem remover um recurso do qual outro dependa. A ordem abaixo é,
portanto, a única válida para um sistema novo.

## Passo 0 — identidade (orgid), antes de qualquer recurso

Antes do forger, é preciso existir **identidade**: uma **organização** e uma **conta dona** com papel
de administrador (via [orgid](orgid/README.md), tipicamente `POST /open/up/account`). O `name` da
organização é o **`{org}`** usado em todos os caminhos do forger, e os **papéis** atribuídos são o que
o forger exige para autorizar. **Sem org/identidade, o forger não autoriza nada.**

```
0. orgid:  organização + conta dona (papel administrador)   → habilita o {org} e a autorização
```

## A sequência (recursos — forger, sob a organização)

```
1. dbconn        metadados de conexão a um servidor de banco relacional
       │
       ▼
2. database      banco de leitura vinculado ao dbconn
       │
       ├──────────────▶ 3a. project      bounded context (unidade do domínio)
       │                       │
       ▼                       ▼
3b. ───────────────────▶ dataschema       esquema vinculado a project + database
                              │           (âncora do isolamento por tenant)
                              ▼
                       4. entity(s)        projeções → tabelas no banco de leitura
                              │
                              ▼
                       5. model            modelo de domínio publicado no cache
                              │            (agregados, comandos, eventos, transições)
                              ▼
                       6. process          modelo de processo/decisão (opcional)
                              │
                              ▼
                       7. ativar           dataschema: MODELING → RUNNING
                                           (habilita comandos/consultas)
```

> **Estado do dataschema durante o deploy:** os passos **4–5** (entity/model) exigem o dataschema em
> **`MODELING`** (esquema em edição). Concluída a modelagem, **transite para `RUNNING`** (passo 7) — só
> então persistence-crs/persistence-q **interpretam e executam** o modelo. Em `MODELING` a operação não
> ocorre. Ver [forger/dataschema — gate de status](forger/endpoints/dataschema.md#atualizar).

### Passo a passo

| # | Recurso | Pré-requisito | O que produz | O que habilita |
|---|---|---|---|---|
| 1 | **dbconn** | — | Registro de conexão a um servidor de banco relacional. | Criar bancos sob essa conexão. |
| 2 | **database** | dbconn | Um banco de leitura. | Vincular esquemas e projeções. |
| 3a | **project** | — | Um bounded context. Gera identificadores de tenant associados ao contexto. | Agrupar esquemas e modelos do domínio. |
| 3b | **dataschema** | project **e** database | Um esquema isolado por tenant. Gera o `tenant-id`. | Hospedar as projeções; isolar comandos/eventos/consultas. |
| 4 | **entity** | dataschema | Tabela de **projeção** no banco de leitura (após validação e compilação da definição). | Que o estado de um agregado seja materializado para leitura. |
| 5 | **model** | dataschema (tenant) | **Modelo de domínio** publicado no cache distribuído. | Que persistence-crs/es-n/persistence-q saibam quais agregados/comandos/eventos existem. |
| 6 | **process** | project | Modelo de processo/decisão publicado. | Orquestrações e decisões (quando o sistema usa processos). |
| 7 | **ativar** | entity + model prontos | Transição do dataschema `MODELING → RUNNING` (`PUT` no `status`). | Que persistence-crs/persistence-q **interpretem/executem** o modelo (operação). |

> **Isolamento:** o `tenant-id` gerado no passo 3b é a chave que isola, dali em diante, o processamento
> de comandos, eventos e projeções desse sistema — no serviço de comandos e no banco (leitura e escrita).

## Estado final

Concluídos os passos 1–5 (6 é opcional) **e ativado o dataschema (passo 7, `RUNNING`)**:

- as **tabelas de projeção** existem no banco de leitura;
- o **modelo de domínio** está publicado no cache distribuído;
- o `tenant-id` está ativo;
- o **dataschema está em `RUNNING`** (sem isso, comandos/consultas não são interpretados).

A partir daí, o sistema aceita **comandos** (persistence-crs) e **consultas** (persistence-q), e o
**es-n** mantém as projeções atualizadas. Ver [arquitetura](01-arquitetura.md).

## Ciclo de vida posterior

Cada recurso admite **leitura**, **atualização** e **remoção** (esta última bloqueada enquanto houver
dependentes). Reenviar um **model** ou um **process** **substitui** o publicado (semântica de
sobrescrita). Detalhes por endpoint em [forger](forger/README.md).
