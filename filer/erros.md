# filer · catálogo de erros

> Envelope e códigos. Guia: [README.md](README.md).

## Envelope
Em erro, a resposta tem o **status HTTP** e um **código de erro textual** (formato `Error 091XX`), sem
envelope JSON.

## Códigos HTTP

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `200` | OK | Operação concluída (upload/download/list/delete). | — |
| `400` | Requisição inválida | Chave ausente/fora do padrão; extensão não permitida; corpo vazio (upload); prefixo curto (list/delete). | Corrigir a chave/corpo conforme o padrão e a whitelist. |
| `401` | Não autenticado | Token ausente/expirado/inválido. | Enviar `Authorization` válido. |
| `403` | Não autorizado | Sem permissão de leitura/escrita no agregado; `tenant-id` fora do escopo. | Usar credencial com o papel adequado no agregado/tenant. |
| `404` | Não encontrado | Arquivo específico inexistente (download; delete de chave completa). | Conferir a chave; em listagem/lote, vazio retorna `200`/`[]`. |
| `413` | Muito grande | Arquivo acima do limite (≈10 MB). | Reduzir o tamanho do arquivo. |
| `500` | Erro interno | Falha no armazenamento de objetos. | Retentar; se persistir, escalar. |

## Notas
- **Limite de tamanho** (≈10 MB) e **whitelist de extensões** são validados **antes** do armazenamento.
- **Lote** (prefixo) não retorna `404`: ausência de correspondências é `200` com `deleted: 0` / `[]`.
- Autorização por **operação**: leitura (download/list) vs escrita (upload/delete), conforme o controle
  de acesso do agregado no modelo do tenant.
