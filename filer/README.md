# filer — guia do serviço

> **Papel:** serviço **transversal de arquivos** — guarda e serve arquivos binários (upload, download,
> listagem, remoção) **vinculados a agregados**. Cada arquivo é um **anexo de um agregado**, identificado
> por uma chave hierárquica que inclui organização, projeto, tipo do agregado, id do agregado e atributo.
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Contents
- O que o filer faz
- Convenção de nome (arquivo ↔ agregado)
- Pré-requisitos do chamador (auth + autorização)
- Índice de endpoints
- Ciclo de vida (upload/download)
- Limites e tipos
- Pontos de coordenação
- Pitfalls (checklist do agente)

---

## O que o filer faz

Operações **CRUD de arquivos** sobre um **armazenamento de objetos** (abstraído). O cliente refere-se a
um arquivo **apenas pelo nome** (chave hierárquica); o mapeamento físico (isolado por tenant) é interno.
Quatro operações: **upload**, **download**, **listar**, **remover**.

## Convenção de nome (arquivo ↔ agregado)

O nome do arquivo **identifica o arquivo no nível do agregado**:

```
{org}-{project}-{entity}-{entityId}-{attribute}.{ext}
```

| Parte | Significado |
|---|---|
| `org` | organização (mesma do `{org}` do forger) |
| `project` | projeto/contexto |
| `entity` | **tipo do agregado** (a entidade/projeção) |
| `entityId` | **id do agregado** (o `aggregateid`) |
| `attribute` | **atributo** do agregado ao qual o arquivo pertence |
| `ext` | extensão (tipo do arquivo) |

Consequências (modelo mental):
- O **agregado vincula o arquivo** — por isso `org` e `project` são conhecidos a partir da chave.
- Um **agregado pode ter vários arquivos**; o convencional é **classificá-los por atributo** (um mesmo
  agregado com vários arquivos, distribuídos entre seus atributos, e possivelmente vários por atributo).
- Logo, `filer` conecta-se à mesma taxonomia do **forger** (entities) e do **persistence-crs**
  (`aggregateid`). Ver [conceitos — do agregado à projeção](../02-conceitos.md#do-agregado-à-projeção-derivação-e-implantação).

> **Listar/remover** aceitam **prefixos parciais** da chave (ex.: `…-entity-`, `…-entityId-`,
> `…-entityId-attribute`) — operando sobre **todos** os arquivos que casam (lote). **Upload/download**
> exigem a chave **completa** (com extensão).

## Pré-requisitos do chamador (auth + autorização)

- **Cabeçalhos:** `Authorization` (token de acesso, esquema portador) **e** `X-Tenant-Id` (tenant).
- **Autorização (RBAC por entidade):** o serviço verifica, no modelo do tenant, o **controle de acesso**
  do agregado para a operação — **leitura** (download/listar) ou **escrita** (upload/remover). Sem o
  papel exigido → `403`. O `tenant-id` precisa pertencer ao portador.

## Índice de endpoints

> Base: `/api/file`. Apenas endpoints autenticados são documentados.

| Operação | Método · Path | Documento |
|---|---|---|
| Upload | `POST /api/file/upload` | [endpoints/upload.md](endpoints/upload.md) |
| Download | `GET /api/file/download` | [endpoints/download.md](endpoints/download.md) |
| Listar | `GET /api/file/list` | [endpoints/listar.md](endpoints/listar.md) |
| Remover | `DELETE /api/file/delete` | [endpoints/remover.md](endpoints/remover.md) |

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md). Contrato de fio: [openapi.yaml](openapi.yaml).

## Ciclo de vida (upload/download)

1. **Auth + tenant** — valida `Authorization` e `X-Tenant-Id`.
2. **Validação da chave** — o nome deve seguir o padrão (5 partes + extensão); extensão na lista
   permitida; corpo não-vazio (upload). Falha → `400` (ou `413` se exceder o tamanho).
3. **Autorização** — confere o controle de acesso do **agregado** (entity) para a operação
   (leitura/escrita), conforme o modelo do tenant. Falha → `403`.
4. **Efeito** — grava/lê/lista/remove no armazenamento de objetos (isolado por tenant).
5. **Resposta** — `200` (conteúdo/relatório); download devolve o binário; ausência → `404`.

## Limites e tipos

- **Tamanho por arquivo:** limite da ordem de **10 MB** (excedido → `413`).
- **Extensões permitidas (whitelist):** documentos (`pdf, doc, docx, xls, xlsx, ppt, pptx, txt, csv`),
  imagens (`jpg, jpeg, png, gif, bmp, tiff`), arquivos (`zip, rar, 7z`), mídia (`mp4, avi, mov, wmv,
  mp3, wav`). Fora da lista → rejeitado (`400`).

## Pontos de coordenação

- **CP-11** — um arquivo é **anexo de um agregado**: a chave amarra `org/project/entity/entityId/attribute`
  à taxonomia do forger (entities) e do persistence-crs (`aggregateid`); a **autorização** usa o
  **controle de acesso do agregado** no modelo do tenant. Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Enviar **sempre** `Authorization` **e** `X-Tenant-Id`.
- [ ] **Upload/download:** chave **completa** com extensão. **Listar/remover:** prefixo parcial = **lote**.
- [ ] Respeitar **tamanho** (≈10 MB) e **extensão** (whitelist) — senão `413`/`400`.
- [ ] A operação é autorizada pelo **controle de acesso do agregado** (entity) — leitura vs escrita.
- [ ] `entityId` é o **id do agregado** (`aggregateid`); use o mesmo da projeção/comando.
- [ ] Vários arquivos por agregado são esperados — organize-os por **atributo** na chave.
