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

## Nota
`204` **não** é erro: significa "nenhum resultado". A consulta é idempotente e segura para repetição.
