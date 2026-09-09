# persistence-q · catálogo de erros

> Envelope e categorias de erro. Guia: [README.md](README.md).

## Códigos HTTP

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `400` | Requisição inválida | JSON malformado; estrutura de critério inválida. | Corrigir o corpo (modo array/object, um rótulo por item). |
| `401` / `403` | Não autorizado | Credencial inválida; `tenant-id` fora do escopo do usuário. | Usar credencial/tenant corretos. |
| `510` | Falha de execução | Predicado/identificador não reconhecido; falha ao ler a projeção. | Usar o vocabulário do tenant; conferir a projeção/entity. |

## Categorias (para diagnóstico)

| Categoria | Significado | Correção |
|---|---|---|
| Validação | Cabeçalho/payload malformado. | Corrigir a requisição. |
| Autorização | Tenant fora do escopo. | Ajustar credencial/tenant. |
| Provisionamento de tenant | Tenant não provisionado / esquema indisponível. | Concluir o deploy (forger). |
| Busca de modelo | Identificador/predicado ausente no modelo. | Usar o vocabulário provisionado. |
| Execução de consulta | Falha ao traduzir/executar o critério sobre a projeção. | Revisar predicados e a definição da entity. |
| Infraestrutura | Falha de banco/conexão/cache. | Retentar; se persistir, escalar. |
| Desconhecida | Não classificada. | Escalar com contexto. |

## Recusas do recorte de leitura por titular

Valem **apenas** para entity que declara recorte; entity sem a declaração não produz nenhuma delas.

| HTTP | Quando | Correção |
|---|---|---|
| `403` | A consulta chega **sem credencial**, ou com credencial que não traz identificação do titular. | Consultar com credencial de usuário; o recorte não tem contra quem comparar. |
| `510` | A entity recorta por um atributo **que não existe** no modelo dela. | Corrigir a declaração no forger. |
| `510` | A entity recorta por **coluna de metadado** (`loguser`, `logversion`). | Usar um atributo de titular. `loguser` registra **quem escreveu** a linha, não de quem ela é. |
| `510` | A entity recorta para um **papel que não está** em `accessControl.read`. | Fazer as duas listas concordarem — normalmente é erro de digitação no nome do papel. |

> As três de `510` são erro de **modelo**, não de requisição: a mesma consulta falha para todo mundo até
> a declaração ser corrigida. Elas existem porque o silêncio, aqui, seria pior — um papel escrito errado
> não recortaria nada e a tabela inteira sairia com `200`.

## Nota
`204` **não** é erro: significa "nenhum resultado". A consulta é idempotente e segura para repetição.
