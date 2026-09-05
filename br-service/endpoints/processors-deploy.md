# br-service · endpoints · implantar processadores

> Publica os **processadores de uma organização** a partir de um arquivo compactado, e desfaz a última
> publicação. Exige **duas credenciais ao mesmo tempo**. Guia: [../README.md](../README.md).

> ⚠️ Este endpoint publica **código que o serviço vai executar**. Quem consegue publicar numa organização
> passa a decidir o que roda nas regras de negócio dela. Trate as credenciais como tal.

## Contents
- Requisição
- Formato do arquivo
- Como montar o pacote (inclui **não perder o que não deveria sair** e **não deixar arquivo inalcançável**)
- O que é recusado
- Resposta
- Erros
- Desfazer
- Exemplos

---

## Requisição

```
POST /v3/brservice/processors/deploy/{org}
```

### Variável de caminho

| Variável | Formato | O que é |
|---|---|---|
| `{org}` | minúsculas, dígitos, `-` e `_`; começa por letra ou dígito; até 64 caracteres | a organização — **primeiro segmento da rota** e pasta raiz dos processadores dela |

Fora desse formato o pedido é recusado. Atenção a **maiúsculas**: nomes de organização são minúsculos, e
`Clubflow` não é o mesmo que `clubflow`.

### Corpo

O **arquivo compactado cru** — não é formulário nem `multipart`, e não vai em campo de upload.

| Item | Valor |
|---|---|
| Corpo | os bytes do `.zip` |
| `Content-Type` | `application/zip` |
| Tamanho máximo | **5 MB** |

> **O `413` não vem deste serviço, e não vem em JSON.** Pacote acima do teto é barrado na borda da
> plataforma, antes de chegar ao br-service, e a resposta é uma **página HTML** de erro da borda — não o
> corpo `{status, mensagem, tipo}` dos erros daqui. **Quem faz parse esperando JSON quebra nesse ponto**;
> trate `413` pelo código, não pelo corpo. Se o seu pacote encostar no limite, o caminho é reduzi-lo
> (processadores são texto; o que costuma inchar um pacote é arquivo que não deveria estar nele).

### Cabeçalhos — os dois são obrigatórios

Faltando qualquer um dos dois, a resposta é `401` e **nada** é publicado.

| Cabeçalho | Obrigatório | Valor | Papel |
|---|---|---|---|
| `X-Processors-Deploy-Credential` | sim | UUID, entregue no provisionamento | autoriza a **operação** — é a credencial do serviço |
| `Authorization` | sim | `Bearer <token de acesso>` | diz **quem** publica e **em que organizações** pode publicar |
| `X-Forger-Credential` | sim | UUID, entregue no provisionamento | exigido pela **borda**, antes de o pedido chegar ao serviço |
| `Content-Type` | recomendado | `application/zip` | o corpo é lido como binário de qualquer forma |

> ⛔ **Nunca envie `X-Tenant-Id` junto com `X-Forger-Credential`.** Os dois no mesmo pedido são recusados
> pela borda com `400` e `tipo: multiple_headers` — *"Cannot have both X-Tenant-Id and X-Forger-Credential
> headers simultaneously"*. Nesta rota vai **só** o `X-Forger-Credential`.

> **Por que três cabeçalhos e não dois.** Esta é uma rota **administrativa**, não de execução de modelo. A
> borda classifica as rotas administrativas e exige `X-Forger-Credential` nelas — `X-Tenant-Id` **não**
> substitui. Faltando ele, a recusa é `401` com `tipo: missing_forger_credential`, e a mensagem diz qual
> cabeçalho falta. O pedido nem chega ao br-service, então as credenciais de deploy e o token não são nem
> avaliados.

As duas credenciais do serviço são **independentes e cumulativas**: a primeira não substitui a segunda. Uma
diz que a operação é permitida neste serviço; a outra, que **este portador** pode publicar **nesta
organização**. É por isso que possuir a credencial não basta para publicar na organização de outro cliente.

O token de acesso é o mesmo obtido no login da plataforma — ver
[autenticação](../../06-autenticacao.md). Sua assinatura é verificada aqui.

A `{org}` da URL precisa estar entre as organizações do portador do token — senão `403`. É o que impede
alguém de publicar na organização de outro cliente. O token tem a **assinatura verificada**; token
forjado, expirado ou com algoritmo trocado é recusado.

## Formato do arquivo

O conteúdo pode vir de dois jeitos, e o serviço aceita os dois:

```
clubflow/pessoal/associado/criar…      ← embrulhado na pasta da organização
pessoal/associado/criar…               ← já com o conteúdo dela na raiz
```

A decisão é do **conjunto**: o prefixo só é removido se **todas** as entradas o tiverem. O caminho de cada
arquivo, sem a extensão, vira o resto da rota — `pessoal/associado/criar` publicado em `clubflow` responde
por `clubflow/pessoal/associado/criar`.

**Substituição é por organização.** Cada envio troca **toda** a árvore daquela organização: o que não
estiver no arquivo enviado **deixa de existir**. As outras organizações não são tocadas. É assim que se
despublica uma regra — enviando um pacote sem ela.

Extensões aceitas: arquivos de processador e `.json` (para dados de apoio). Qualquer outra é recusada.

## Como montar o pacote

**A regra única:** o caminho de cada arquivo **dentro do pacote**, sem a extensão, é o que vem **depois do
nome da organização** na rota. Errar a profundidade não dá erro — publica, e a rota fica diferente da que
o modelo aciona.

Partindo de uma árvore assim, com a organização `clubflow`:

```
clubflow/
└── pessoal/            ← project
    └── pessoal/        ← bounded context
        └── associado/  ← agregado
            ├── criar
            ├── atualizar
            └── encerrar
```

As duas formas de compactar produzem pacotes aceitos — escolha uma:

```
# A) de dentro da pasta da organização (o pacote não a contém)
cd clubflow && zip -r ../clubflow.zip .

# B) de fora, incluindo a pasta da organização
zip -r clubflow.zip clubflow
```

Em A, as entradas ficam `pessoal/pessoal/associado/criar…`; em B, `clubflow/pessoal/pessoal/associado/criar…`.
O serviço reconhece as duas — na B ele remove o prefixo, e só o faz se **todas** as entradas o tiverem.

**O erro que passa despercebido** é compactar do diretório errado, uma pasta acima:

```
# ERRADO: gera "clubflow/clubflow/pessoal/…" quando enviado como forma B
cd .. && zip -r clubflow.zip meu-projeto/clubflow
```

O pacote é aceito, os arquivos são gravados — e a rota vira
`clubflow/meu-projeto/clubflow/pessoal/…`, que **nenhum modelo aciona**. O sintoma é a regra "não rodar"
sem erro nenhum. Confira sempre a lista `rotas` da resposta contra a `br.route` declarada no modelo.

Confira o conteúdo antes de enviar:

```
unzip -l clubflow.zip
```

### Não perder o que não deveria sair

A publicação **substitui a árvore inteira da organização**. Por isso:

- **Monte o pacote a partir da árvore completa da organização**, nunca só dos arquivos que você mexeu.
  Enviar um pacote com um arquivo só **apaga todos os outros** daquela organização.
- Para **despublicar** uma regra, é o contrário: monte a árvore completa **sem** ela.
- Depois de publicar, olhe **`recarga.removidas`** na resposta. Se aparecer ali algo que você não queria
  remover, o pacote estava incompleto — reenvie a árvore completa, ou desfaça pelo `backupId`.

### Não deixar arquivo inalcançável

Um arquivo pode ser gravado com sucesso e mesmo assim **nunca virar rota**. Três causas, em ordem de
frequência:

| Causa | Como aparece |
|---|---|
| **Profundidade errada** no pacote | a rota existe, mas com caminho diferente do que o modelo aciona — aparece em `recarga.adicionadas` com um caminho que você não reconhece |
| **Não exporta uma função de processador** | aparece em `arquivos` e **não** em `recarga.adicionadas` |
| **Extensão não aceita** | recusado com `400`; nada é publicado |

Erro de sintaxe **não** entra nessa lista: é barrado antes, com `400`.

## O que é recusado

Antes de qualquer coisa ser gravada, o pacote inteiro é validado. Falhou uma entrada, **nada** é publicado
— não existe publicação pela metade.

| Recusa | Por quê |
|---|---|
| Entrada com `..` no caminho | escaparia da pasta da organização |
| Caminho absoluto | idem |
| Link simbólico | apontaria para fora da árvore |
| Extensão não aceita | só processador e `.json` entram |
| **Erro de sintaxe** | cada arquivo é **compilado** (sem executar) antes de entrar |
| Arquivo maior que o declarado | defesa contra pacote-bomba |
| Entradas ou bytes acima do teto | idem |
| Dois arquivos com o mesmo destino | ambiguidade |
| Arquivo compactado ilegível | — |

A checagem de sintaxe importa mais do que parece: um processador que não compila seria **descartado em
silêncio** no carregamento, e a rota simplesmente não existiria — sem nenhum aviso.

## Resposta

`200` — publicado:

```json
{
  "org": "clubflow",
  "backupId": "<id para desfazer, ou null se não havia árvore anterior>",
  "substituiuArvoreAnterior": true,
  "arquivos": ["pessoal/associado/criar…"],
  "rotas": ["clubflow/pessoal/associado/criar"],
  "recarga": {
    "total": 12,
    "adicionadas": ["clubflow/pessoal/associado/criar"],
    "removidas": ["clubflow/pessoal/associado/antigo"]
  }
}
```

> Ilustrativo — `<...>` é placeholder, não JSON literal.

- **As rotas entram em vigor na hora**, sem reinício do serviço.
- **`recarga.adicionadas` é delta, não confirmação de recarga.** Lista as rotas **novas** — as que não
  existiam antes deste envio. Republicar o conteúdo de uma rota que já existia **não** aparece ali: um
  envio que só altera regras existentes volta com `adicionadas: []`, e isso é o esperado, não falha. Para
  confirmar que a publicação entrou, use **`recarga.total`** (quantas rotas o serviço atende agora) e a
  lista `rotas`.
- **Para achar arquivo que não virou rota**, compare `arquivos` com `rotas`: o que não exporta uma função
  de processador aparece em `arquivos` e não em `rotas`.
- **`recarga.removidas`** lista o que deixou de existir por não estar no pacote.
- **Guarde o `backupId`** — é o que permite desfazer.

## Erros

| Código | Quando |
|---|---|
| `400` | pacote inválido: caminho suspeito, extensão recusada, erro de sintaxe, teto estourado, corpo vazio, organização com nome inválido |
| `400` **da borda** | `X-Tenant-Id` enviado junto com `X-Forger-Credential` (`tipo: multiple_headers`) |
| `401` **da borda** | `X-Forger-Credential` ausente (`tipo: missing_forger_credential`) — o pedido não chega ao serviço |
| `401` | `X-Processors-Deploy-Credential` ausente/incorreta, ou token ausente, expirado, com assinatura inválida ou algoritmo trocado |
| `403` | o portador do token **não está vinculado** à organização da URL |
| `500` | falha ao trocar a árvore no destino — o estado anterior é restaurado |
| `503` | implantação não provisionada neste ambiente (credencial ou verificação de token ausente) |

Corpo de erro: `{ status, mensagem, tipo }`. Catálogo geral: [../erros.md](../erros.md).

## Desfazer

```
POST /v3/brservice/processors/rollback/{org}/{backupId}
```

Mesmos dois cabeçalhos e a mesma checagem de organização. Repõe a árvore que estava no lugar antes da
publicação identificada por `{backupId}` e recarrega as rotas. Responde com a mesma forma de `recarga`.

### Variáveis de caminho

| Variável | Formato | O que é |
|---|---|---|
| `{org}` | mesmo do deploy | a organização cuja árvore será reposta |
| `{backupId}` | o valor **devolvido no campo `backupId`** da resposta da publicação | identifica qual publicação desfazer |

Sobre o `{backupId}`:

- **Não é inventável** — só funciona o que veio numa resposta de publicação.
- **Pertence a uma organização**: começa pelo nome dela, e usá-lo na URL de outra é recusado. É o que
  impede quem publica numa organização de mexer no histórico de outra.
- Vem `null` quando a publicação **não** substituiu nada (primeira publicação daquela organização) —
  nesse caso não há o que desfazer.
- Desfazer é uma **troca**, não um apagamento: a árvore que estava no ar é guardada antes de ser
  substituída pela anterior.

## Exemplos

Publicar:

```
POST /v3/brservice/processors/deploy/clubflow
X-Processors-Deploy-Credential: <uuid>
X-Forger-Credential: <uuid>
Authorization: Bearer <token>
Content-Type: application/zip

<bytes do arquivo .zip>
```

Com uma ferramenta de linha de comando, o essencial é enviar o **binário cru**:

```
curl -X POST \
  -H 'X-Processors-Deploy-Credential: <uuid>' \
  -H 'X-Forger-Credential: <uuid>' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/zip' \
  --data-binary @clubflow.zip \
  https://<base>/v3/brservice/processors/deploy/clubflow
```

Desfazer a última publicação:

```
POST /v3/brservice/processors/rollback/clubflow/<backupId>
X-Processors-Deploy-Credential: <uuid>
X-Forger-Credential: <uuid>
Authorization: Bearer <token>
```

Depois de publicar, confirme pela [consulta de log](logs.md) que as execuções da rota nova estão saindo com
`outcome: success` — o `erroArquivo` aponta o arquivo exato quando não estiverem.
