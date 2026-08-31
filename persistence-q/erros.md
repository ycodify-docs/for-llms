# persistence-q · catálogo de erros

> Envelope e categorias de erro. Guia: [README.md](README.md).

## Códigos HTTP

| HTTP | Significado | Causas típicas | Correção |
|---|---|---|---|
| `400` | Requisição inválida | JSON malformado; estrutura de critério inválida. | Corrigir o corpo (modo array/object, um rótulo por item). |
| `401` / `403` | Não autorizado | Credencial inválida; `tenant-id` fora do escopo do usuário. | Usar credencial/tenant corretos. |
| `510` | Falha de execução | Predicado/identificador não reconhecido; **rótulo que não existe no modelo do tenant** (ver nota abaixo); falha ao ler a projeção; **modelo do tenant ausente do cache** (ver nota abaixo). | Usar o vocabulário do tenant; conferir a projeção/entity; se a mensagem falar em **modelo não encontrado na cache** — ou, em versão antiga, em `X-Tenant-Id` — ver a nota. |

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

## Nota — `510` de **rótulo que não existe no modelo** (não confundir com o de baixo)

Rótulo que não corresponde a nenhuma entidade/projeção provisionada para o tenant:

```
510 · Entidade '<rótulo>' não existe no modelo do tenant '<tenant-id>'. Na consulta, o rótulo deve ser
o nome da entidade/projeção provisionada para o tenant; confira o nome ou republique o modelo.
```

**Distinção que importa:** aqui o **modelo do tenant está carregado** — o que falta é *aquele nome* dentro
dele. Já a nota seguinte é o caso em que o modelo **inteiro** não está disponível. Sintoma parecido,
correção diferente: aqui, corrigir o rótulo (ou publicar a entidade); lá, republicar o modelo.

> **Mudou em 2026-08-31.** Antes esta falha vazava uma exceção interna de biblioteca como mensagem de
> usuário (`JSONObject["<rótulo>"] not found`), sem dizer o que fazer. Se o seu cliente casava aquele
> texto, **ajuste** — o código HTTP (`510`) não mudou.

## Nota — `510` de **modelo do tenant ausente do cache**

Acontece quando o **tenant é válido** mas o **modelo dele não está no cache** — por exemplo, um tenant novo
cujo modelo ainda não foi publicado, ou cuja entrada expirou.

**Duas mensagens, conforme a versão do serviço:**

| Mensagem | O que significa |
|---|---|
| `Modelo do tenant não encontrado na cache (chave '<chave>'). O X-Tenant-Id foi reconhecido; o que falta é o modelo. Publique/republique o modelo do tenant (dataschema MODELING -> RUNNING).` | texto **atual** — nomeia a causa e a ação |
| `Falha crítica. X-Tenant-Id não reconhecido.` | texto **anterior**, ainda possível em ambiente que não tenha recebido a versão nova. **Enganoso:** aponta para o cabeçalho, mas a causa é a mesma — o modelo ausente |

**Nos dois casos, a ação é a mesma** (e não é mexer no `X-Tenant-Id`): confirme que o modelo do tenant foi
publicado e que o dataschema está em `RUNNING` (ver
[forger/dataschema](../forger/endpoints/dataschema.md#atualizar)). Republicar (`MODELING` → `RUNNING`) repõe
a entrada.
