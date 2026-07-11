# filer · exemplos anotados

> Dados genéricos. Todas as requisições levam `Authorization` + `X-Tenant-Id`. Guia: [README.md](README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não conteúdo literal.

## 1. Upload (anexo de um atributo de um agregado)

```
POST /api/file/upload?filename=acme-vendas-pedido-42-comprovante.pdf
Headers: Authorization: ..., X-Tenant-Id: <tenant-id>, Content-Type: application/octet-stream
Body: <bytes do PDF>
→ 200  "File uploaded successfully. ... Attribute: comprovante  Size: <bytes>"
```

Anexa um PDF ao **atributo `comprovante`** do **agregado `pedido` de id `42`** (org `acme`, projeto `vendas`).

## 2. Download (chave completa)

```
GET /api/file/download?filename=acme-vendas-pedido-42-comprovante.pdf
Headers: Authorization: ..., X-Tenant-Id: <tenant-id>
→ 200  (binário)  Content-Disposition: attachment; filename="comprovante.pdf"
```

## 3. Listar todos os arquivos de um agregado (prefixo)

```
GET /api/file/list?filename=acme-vendas-pedido-42-
→ 200 [ "<ref .../42/comprovante.pdf>", "<ref .../42/nota.pdf>", "<ref .../42/foto.jpg>" ]
```

## 4. Remover por atributo (lote)

```
DELETE /api/file/delete?filename=acme-vendas-pedido-42-comprovante
→ 200 { "deleted": 2, "prefix": "<ref>", "files": [ "<...comprovante.pdf>", "<...comprovante.png>" ] }
```

## 5. Erros comuns

```
POST /api/file/upload?filename=arquivo.pdf            → 400  (chave fora do padrão de 5 partes)
POST /api/file/upload?filename=acme-vendas-pedido-42-x.exe → 400  (extensão fora da whitelist)
POST /api/file/upload  (12 MB)                        → 413  (acima do limite)
GET  /api/file/download?filename=acme-vendas-pedido-42-inexistente.pdf → 404
```

> `entityId` (`42`) é o **id do agregado** (`aggregateid`). Um agregado pode ter vários arquivos,
> convencionalmente classificados por **atributo**. Ver [README](README.md) e [coordenação](../coordenacao.md).
