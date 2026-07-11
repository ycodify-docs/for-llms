# forger — guia do serviço

> **Papel:** implanta **todos os recursos** de um sistema. É o **primeiro** serviço da plataforma: nada
> opera antes que forger crie os recursos e publique os modelos.
> Pré-requisitos de leitura: [conceitos](../02-conceitos.md), [fluxo de deploy](../03-fluxo-de-deploy.md).

## Contents
- Pré-requisitos do chamador
- Conceitos próprios
- Índice de endpoints (matriz de cobertura)
- Ciclo de vida geral de uma requisição
- Pontos de coordenação
- Pitfalls (checklist do agente)

---

## Pré-requisitos do chamador

- **Autenticação:** toda requisição exige o cabeçalho `Authorization`.
- **Autorização:** o usuário precisa ter, na organização (`{org}`) alvo, o papel de **administrador** ou
  de **engenheiro**. Sem isso → `403`.
- **Ordem:** respeitar a sequência de deploy (um recurso não pode ser criado sem o seu "pai"). Ver
  [fluxo de deploy](../03-fluxo-de-deploy.md).

## Conceitos próprios

forger administra **recursos** (não agregados de domínio). Cada recurso tem ciclo de vida
**criar → ler → atualizar → remover**, com **versão otimista** na atualização (campo `logversion`) e
**remoção bloqueada** enquanto houver dependentes. Recursos:
`dbconn`, `database`, `project`, `dataschema`, `entity`, `model`, `process`, `email`
(ver [conceitos](../02-conceitos.md)).

### Regras de nome dos recursos
Nomes são **normalizados para minúsculas** e seguem formato e **limite de comprimento** exatos:

| Recurso | Caracteres aceitos | Padrão | Máx. caracteres |
|---|---|---|---|
| **database** (`dbsqlname`) | só letras minúsculas | `^[a-z]+$` | — (sem limite definido) |
| **project** (`name`) | só letras minúsculas | `^[a-z]+$` | **16** |
| **dataschema** (`name`) | minúsculas, dígitos e `_`; **começa com letra** | `^[a-z][a-z0-9_]*$` | **16** |
| **entity** (nome) | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| **atributo** (nome) | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| **associação** (nome) | só letras minúsculas (sem dígitos/`_`) | `^[a-z]+$` | **24** |
| **superentidade** (nome) | só letras minúsculas | `^[a-z]+$` | **24** |

Violar o padrão ou o limite → `400` com a indicação do nome rejeitado.

### Nomes reservados e atributos gerenciados
- **Nomes reservados** (não podem nomear entity/atributo/associação): `id`, `logversion`, `logrole`,
  `loguser`. Usá-los → `400`.
- Toda projeção recebe automaticamente atributos de controle (identidade e versionamento/auditoria,
  os nomes reservados acima). São **injetados e mantidos pela plataforma** — **não** os declare na
  definição da entity nem os envie em payloads de escrita; fazê-lo causa rejeição.

### Ciclo do `tenant-id`
O `tenant-id` é **gerado na criação do dataschema** (não é escolhido pelo chamador). Para usá-lo nos
demais serviços, **obtenha-o** lendo o recurso/listagem correspondente — trate-o como **opaco** (não
interprete sua estrutura).

## Índice de endpoints (matriz de cobertura)

> Caminhos relativos à raiz do serviço. `{org}` = organização; demais chaves entre `{}` são identificadores.
> Apenas endpoints **autenticados** são documentados. Endpoints internos sem autenticação **não** constam
> desta documentação pública.

| Recurso | Endpoint | Documento |
|---|---|---|
| **dbconn** | `POST/GET/PUT/DELETE /org/{org}/dbconn[/{id}]` | [endpoints/dbconn.md](endpoints/dbconn.md) |
| **database** | `POST/GET/PUT/DELETE /org/{org}/dbconn/{dbconnId}/database[/{id}]` (+ busca por nome) | [endpoints/database.md](endpoints/database.md) |
| **project** | `POST/GET/PUT/DELETE /org/{org}/project[/{project}]` | [endpoints/project.md](endpoints/project.md) |
| **dataschema** | `POST/GET/PUT/DELETE /org/{org}/project/{project}/database/{databaseId}/dataschema[/{dataSchema}]` | [endpoints/dataschema.md](endpoints/dataschema.md) |
| **entity** | validar, criar, ler, listar, atualizar, +atributo, +associação, −atributo, −associação, remover, analisar | [endpoints/entity.md](endpoints/entity.md) |
| **model** | `POST/GET/DELETE .../tenant/{tenantId}/model` + listar | [endpoints/model.md](endpoints/model.md) |
| **process** | `POST/GET/DELETE /org/{org}/project/{project}/process` | [endpoints/process.md](endpoints/process.md) |
| **email** | `POST/GET/PUT/DELETE /org/{org}/project/{project}/email[/{id}]` | [endpoints/email.md](endpoints/email.md) |

Catálogo de erros: [erros.md](erros.md). Exemplos anotados: [exemplos.md](exemplos.md).

> **Gramáticas BNF dos payloads:** [`bnfs/`](bnfs/README.md) traz gramáticas **BNF** que descrevem
> formalmente a estrutura JSON esperada por cada endpoint de criação/atualização (dbconn, database,
> project, dataschema, entity, atributo, associação, update). Um agente pode consultá-las para
> **construir e validar** o corpo de uma requisição ao forger sem ambiguidade.

## Ciclo de vida geral de uma requisição

Vale para todos os endpoints de forger (variações específicas em cada documento):

1. **Autenticação/autorização** — valida `Authorization` e o papel na organização. Falha → `403`.
2. **Validação de entrada** — normaliza chaves de caminho (minúsculas), valida o corpo (campos
   obrigatórios, formatos restritos de nome). Falha → `400`.
3. **Verificação de dependências/consistência** — confere que os recursos "pais" existem e que a
   operação não orfana nem conflita. Conflito de versão → `409`; ausência → `404`.
4. **Efeito** — conforme o recurso:
   - recursos de infraestrutura (`dbconn`, `database`, `dataschema`): grava metadados **e** materializa
     o efeito físico no banco relacional (criar banco/esquema), com rollback se o efeito físico falhar;
   - `entity`: **compila** a definição (léxica → sintática → semântica → representação intermediária →
     DDL) e aplica a DDL no banco de leitura, de forma transacional;
   - `model`/`process`: **publica** o documento no cache distribuído (semântica de sobrescrita).
5. **Resposta** — `201` (criação), `200` (leitura/atualização/remoção), `204` (sem conteúdo). Erros
   conforme [erros.md](erros.md).

## Pontos de coordenação

- **CP-1** — o **model** publicado por forger é lido por persistence-crs, es-n e persistence-q.
- **CP-2** — as **entities** criadas por forger viram as tabelas de projeção que persistence-q consulta.

Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Criar recursos **na ordem**: dbconn → database → (project, dataschema) → entity → model.
- [ ] `dataschema` exige **project e database** já existentes; ele gera o `tenant-id` do sistema.
- [ ] Publicar **model** só após as **entities** existirem (as projeções precisam de tabela).
- [ ] Em atualização (`PUT`), enviar o `logversion` atual; divergência → `409` (alguém atualizou antes).
- [ ] Não tentar remover um recurso com dependentes — a remoção é bloqueada.
- [ ] Reenviar **model**/**process** **sobrescreve** o publicado; é a forma de versionar.
- [ ] Nomes de recursos seguem formatos restritos (minúsculas, comprimento limitado) — validar antes.
