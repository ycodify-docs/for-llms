# Gramáticas BNF dos payloads do forger

> Gramáticas **BNF** (Backus–Naur Form) que descrevem **formalmente** a estrutura JSON esperada por
> endpoints de criação/atualização do forger. Úteis para um agente **gerar e validar payloads**
> corretos antes de chamar a API. Guia do serviço: [../README.md](../README.md).

## Para que servem
- **Referência formal** da forma exata do corpo de cada requisição (campos, tipos, aninhamento).
- **Geração de payload** por agentes: a gramática elimina ambiguidade sobre o que é válido.
- **Validação** prévia de uma requisição antes de enviá-la.

> As gramáticas são a fonte **formal**; a descrição em prosa de cada endpoint está em
> [../endpoints/](../README.md#índice-de-endpoints-matriz-de-cobertura). Em dúvida sobre limites de
> nome/restrições, ver a seção de regras no [guia do forger](../README.md).

## Arquivos

| Gramática | Endpoint | Documento em prosa |
|---|---|---|
| [`create-dbconn-request.bnf`](create-dbconn-request.bnf) | `POST /org/{org}/dbconn` | [dbconn](../endpoints/dbconn.md) |
| [`create-database-request.bnf`](create-database-request.bnf) | `POST .../dbconn/{id}/database` | [database](../endpoints/database.md) |
| [`create-project-request.bnf`](create-project-request.bnf) | `POST /org/{org}/project` | [project](../endpoints/project.md) |
| [`create-dataschema-request.bnf`](create-dataschema-request.bnf) | `POST .../database/{id}/dataschema` | [dataschema](../endpoints/dataschema.md) |
| [`create-entity-request.bnf`](create-entity-request.bnf) | `POST .../dataschema/{ds}/entity` | [entity](../endpoints/entity.md) |
| [`update-entity-request.bnf`](update-entity-request.bnf) | `PUT .../entity/{entity}` | [entity](../endpoints/entity.md) |
| [`create-attribute-request.bnf`](create-attribute-request.bnf) | `POST .../entity/{entity}/attribute` | [entity](../endpoints/entity.md) |
| [`create-association-request.bnf`](create-association-request.bnf) | `POST .../entity/{entity}/association` | [entity](../endpoints/entity.md) |

## Notas
- Os exemplos de **endereço de banco**, **usuário** e **valores** nas gramáticas são **genéricos**
  (ex.: `db.example.internal`, `admin_example`) — substitua pelos do seu ambiente.
- As gramáticas descrevem o **contrato de entrada**; o comportamento e os efeitos de cada endpoint
  estão no documento em prosa correspondente.
