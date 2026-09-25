# CHANGELOG — documentação pública (for-llms)

> Histórico de revisões desta documentação. Datas em formato `AAAA-MM-DD`.

## 1.45 — 2026-09-25

- **`computed` volta a nunca mandar `null` — e a 1.44 fica desdita.** A 1.44 fazia o campo que aceita nulo
  receber `null` explícito. Medido depois por um sistema consumidor, direto no persistence-crs, o `null`
  explícito **quebra a leitura**: na criação a projeção **não materializa a linha** (o agregado existe, a
  consulta não o acha), e na transição o write model grava `NULL` enquanto a projeção **mantém o valor
  anterior**. Só valor explícito não nulo deixa os dois lados iguais. O marcador passa a ser sempre não
  nulo; no texto, `""` se o campo aceita nulo e `"computed"` se é obrigatório (`bff/README.md`
  §Atributo calculado pelo br, com a tabela medida). **A 1.44 nunca chegou ao ar** — o deploy foi
  segurado antes. A regra do processor fica uma só: nunca devolver `null`; para limpar, o vazio do tipo.

## 1.44 — 2026-09-25

- **`computed`: o campo que aceita nulo passa a receber `null` explícito, não um valor do tipo**
  (`bff/README.md` §Atributo calculado pelo br). Até a 1.43, todo campo calculado recebia `"computed"`,
  `0`, `1970-01-01`… — e, como o motor **ignora o `null`** que o processor devolve, um processor que anula
  de propósito um bloco opcional gravaria o marcador em toda operação comum. Achado pelo `clubflow` ao
  avaliar o impacto da 1.43, antes de qualquer declaração em produção. O `null` explícito sobrevive até
  a mescla e é substituível (lido no motor pelo `interpreter`). O obrigatório segue com o valor do tipo.
  A página passa também a dizer que **devolver `null` não limpa campo nenhum**, e que value object
  ausente não é criado pelo marcador.

## 1.43 — 2026-09-25

- **Atributo calculado pelo br — `computed`** (`bff/README.md`, nova seção; `shell/contrato-miolo.md`).
  O modelo passa a poder declarar, ao lado dos comandos do agregado, quais atributos o processor de
  regra de negócio **preenche** — escalares e campos de value object. A capacidade os marca com o papel
  novo `computed`, o miolo genérico não os desenha, e o BFF os envia **sempre** com um marcador do tipo,
  que o processor sobrescreve. Antes, a chave calculada não chegava ao processor — a mescla dele só
  substitui chave presente, e o BFF descarta `""`/`null` —, e a única saída era o cliente digitar um
  placeholder. Declaração inválida, inclusive em comando sem `br.route`, recusa o comando com `500`.
  ⚠️ **Não é a forma definitiva:** se o processor não devolver o campo, o marcador é gravado sem aviso.
  Pedido do `conceptnatal`.

> **Quem já integrou não precisa mexer em nada:** sem a declaração, nenhum atributo é `computed` e tudo
> se comporta como antes. Consumidor que trate `role` como conjunto fechado de quatro valores precisa
> aceitar o quinto.

## 1.42 — 2026-09-23

- **As chaves de cache internas saem da doc MASCARADAS** — por ordem do dono, hoje. Onde a página
  precisava falar delas, o prefixo interno virou `<prefixo-interno>` e **o aviso de que o formato está
  mascarado vem colado à apresentação da chave**, não em rodapé: quem lê não pode sair daqui achando que
  copiou um literal. O **sentido** fica inteiro, porque o que a doc afirma nunca foi o valor — é a
  **distinção** entre as duas chaves (uma termina em `:wm`, o write model que a capacidade lê; a outra
  guarda a spec de entidades, que o motor usa) e o fato de que **o que apaga uma não apaga a outra**.
  Atingidos: `bff/README.md` §Capacidade e `forger/endpoints/model.md`, no corpo do `201` — ali a nota diz
  com todas as letras que **a resposta real traz a chave completa, e é de lá que se lê**, não da doc.

- **Nome de classe e de método também saíram**, pela mesma razão e no mesmo lugar: `bff/README.md` passa
  a dizer *"o controlador de modelo do forger manda o serviço de cache remover a entrada daquele tenant"*.
  Implementação não é contrato; quem integra não precisa do nome, e quem mantém o serviço não lê esta doc
  para descobri-lo.

- **`orgid/endpoints/ua-papel.md`: o terceiro padrão de rota liberada também está mascarado.** A passagem
  continua explicando o que precisava explicar — por que o orgid **sozinho** responde `401` num endpoint
  documentado como público —, dizendo que além de `/open/**` e `/z/**` **existe** um terceiro padrão
  interno, sem publicar qual. A config de segurança deixou de ser nomeada por classe.

> **O que NÃO mudou:** nenhum comportamento, nenhum contrato, nenhum endpoint. É a mesma doc dizendo as
> mesmas coisas com menos valor interno exposto. Quem já integrou não precisa mexer em nada.

## 1.41 — 2026-09-23

> As três correções abaixo viviam numa seção *"Histórico de correções"* dentro da página do BFF. A
> página passa a dizer só **como é**; o que ela corrigiu, e quando, mora aqui. As datas são as das
> correções, não a desta revisão.

- **2026-09-13 · `valueObjects` passou a existir na capacidade.** Antes disso o comando aceitava o
  campo no envio e a capacidade não o descrevia: quem montava tela a partir dela concluía que o campo
  não existia, sem aviso, e o miolo genérico **não conseguia disparar comando com value object
  obrigatório**, por montar o formulário só de `attributes`. Medido de fora, comparando o modelo
  publicado com a capacidade devolvida.

- **2026-09-10 · o modelo publicado nunca teve TTL** — e a doc afirmou o contrário por catorze dias.
  A afirmação **era verdadeira quando foi escrita**: até 2026-08-31 o forger gravava com prazo de 24h
  configurável. Naquele dia ele passou a gravar sem prazo e a propriedade sumiu; esta fatia não foi
  atualizada junto. **O risco é estrutural e não se resolve conferindo melhor: doc de comportamento
  alheio envelhece quando o dono do comportamento muda sem avisar quem documentou.** A primeira
  correção ainda errou de chave, mandando investigar o bracket de um modelo cuja remoção não passa por
  ali — são **dois serviços de cache distintos**: um grava o **write model** (o que o BFF lê), o outro
  grava o **read model** (o que o bracket remove), e confundi-los é o que levou a investigar o lado errado. **Parágrafo cuja receita sempre dá certo — aqui, republicar —
  não avisa quando a explicação está errada.**

- **2026-09-13 · a regra do `readProjection` deixou de copiar a do `scope` de linha.** A regra antiga
  era *"vários papéis, um deles fora da declaração → lê tudo"*, correta **quando o outro papel de fato
  lê a entity** — e o BFF não tem como saber isso, porque quem concede leitura é o `accessControl.read`
  do `_conf`, que ele não enxerga. Medido com conta real: um usuário `[VISITANTE, RECEPCIONISTA]`
  recebia a ficha inteira **com CPF**, porque `VISITANTE` não estava na declaração, embora não tivesse
  leitura nenhuma naquela entity. Como quase todo usuário acumula papéis, o controle era praticamente
  inerte. A regra atual **falha fechando**.

## 1.40 — 2026-09-23

- **BFF — `serverStamp` é o `whenAttribute` do evento, e não mais o sufixo do nome.** Até esta revisão
  a doc descrevia `serverStamp` só como "carimbo do servidor", e a implementação classificava por
  `Timestamp` com nome terminado em `em` — o que retirava do comando atributos que o modelo declarava
  como dados, inclusive `nullable:false`. Quem tinha contorno para isso (renomear o atributo, ou
  declará-lo como `whenAttribute` de um evento só para o valor ser aceito) não precisa mais dele.
  Páginas: [bff](bff/README.md), [contrato do miolo](shell/contrato-miolo.md).

- **BFF — as quatro rotas de arquivo anexo de agregado passam a existir**: `/session/files/upload`,
  `/download`, `/list` e `/delete`, proxy do [filer](filer/README.md) pelo cookie de sessão. Página:
  [bff](bff/README.md#arquivos-anexos-de-agregado-sessionfiles).

- **Teto de arquivo: a página do BFF diz 3 MiB; a do filer ainda anuncia "≈10 MB".** O valor que vale
  para quem passa pelo BFF é 3 MiB — é o `max-file-size` do próprio serviço. A página do filer será
  reconciliada por quem responde por ela.

## 1.39 — 2026-09-23

- **O recorte de leitura passa a ter DUAS dimensões, e elas são independentes.** `rows.by` responde
  quais **linhas** são suas; `attributes`, quais **colunas** você vê. Vale nos dois lugares: no
  `_conf.accessControl.scope.read` da **entity** ([entity](forger/endpoints/entity.md)) e no
  `roles.scope.read` do **aggregate** ([model](forger/endpoints/model.md)). Declare uma, a outra, ou as
  duas.

- **`rows` deixou de ser obrigatório**, e é isso que destrava o caso real: uma **conta de integração**
  não é titular de linha nenhuma — não existe `rows.by` possível para ela —, então, enquanto
  `attributes` só valesse dentro de um bloco que exigia `rows`, o caso ficava de fora por construção.
  O que essa conta precisa é ver **duas colunas de todas as linhas**, em vez das vinte e quatro.

- **Papel sem nenhuma das duas dimensões passa a ser recusado com `400`.** Recorte que não corta nada
  se lê como proteção e não é. É a única recusa nova.

- **Os nomes em `attributes` NÃO são conferidos na publicação**, de propósito: nome desconhecido o motor
  ignora com aviso, e lista inteira errada devolve resposta vazia. Recusar por typo transformaria erro
  de modelagem em entity indisponível. O contraste com `rows.by` é deliberado — lá o nome **é**
  conferido, porque um `by` errado não devolve menos, devolve **errado**.

- **Vigência: EM PRODUÇÃO desde 2026-09-23T02:38Z**, em `yc-composer:amd64-9884832-74f6`
  (`composer@9884832`, `forger@8b906dd`, `auth@537991a`, `orgid@697f402`). A outra metade — o motor que
  **aplica** o corte — subiu às 02:28:11Z em `tinterpreter:amd64-538ea49-2eca`
  (`ufrn.loco3@22fcb89`, `persistence-crs@40493a0`).

  **As duas metades no ar é o que torna o recurso real:** a publicação aceitar `attributes` sem o motor
  cortar seria proteção declarada e não aplicada. Medido dentro do contêiner em execução, e não no jar
  da máquina de build: o `schema/aggregate-schema.json` que está rodando contém `write-model-v2` e
  `attributes`.

## 1.38 — 2026-09-21

- **O histórico de um agregado tocado por coordenação volta a responder.** O `GET` de histórico desses
  agregados devolvia erro permanente; agora responde `200`. **Os eventos gravados a partir desta imagem
  trazem a autoria; os anteriores vêm sem ela** — a identidade não foi gravada quando eles aconteceram, e
  não há de onde tirá-la. Histórico antigo aparece legível, com alguns eventos sem autor.

- **Todo comando passa a carimbar quem o fez.** O dado do agregado ganha a chave `loguser` nas
  respostas. Ela **não** desce para a projeção — o modelo de leitura segue com as colunas que você
  declarou.

- ⚠️ **A autoria passa a valer na materialização da projeção.** O autor do comando de origem agora
  acompanha a coordenação até a escrita da projeção, e o `accessControl.write` da entity de destino é
  conferido de verdade. **Se o papel do autor não estiver no `write` da entity alvo, a projeção não
  materializa** — vale conferir as coordenações em que quem inicia tem papel diferente do que o destino
  aceita.

- **Campo declarado `nullable: false` passa a ser exigido pelo motor, e não só pelo BFF.** Comando que
  não traga o campo, ou que o traga nulo, é **recusado**. Até aqui a declaração era lida pelo BFF do
  Stager e nada mais: comando que entrasse por outro caminho — coordenação, despacho de comando-alvo,
  chamada direta ao endpoint — era aceito com o campo ausente. ⚠️ Se o seu modelo declara
  `nullable: false` num campo que **o processador de regra de negócio calcula**, esse comando passa a ser
  recusado **antes** de a regra rodar.

- **Falha de projeção por chave divergente deixa de ser silenciosa.** A notificação era descartada sem
  log, sem métrica e sem fila de erro — indistinguível de "nada aconteceu". Agora sai registro com as
  chaves que o modelo tem, contador e erro publicado; e identificador de agregado malformado passa a ser
  descartado com motivo, em vez de derrubar o processamento do lote.

- **Vigência:** em **teste** desde `yc-interpreter:amd64-260921` (campo obrigatório e descarte com
  registro; `yc.cqrs-c@9e2ea51`, `yc.es@0b981db`) e `:amd64-260921b` (histórico, `loguser` e autoria na
  coordenação; `yc.cqrs-c@0f600d3`, `persistence-crs@8ba070f`, `ufrn.loco3@b8a944b`), implantadas em
  2026-09-21. **Produção ainda não** — o interpreter segue em `:amd64-260913c`. A validação do
  metamodelo, que subiu em produção no mesmo dia, está na 1.37 e não se repete aqui.

## 1.37 — 2026-09-21

- **Produção passou a rodar `yc-composer:amd64-260921` (`sha256:fecea707b971`), implantado às
  16:39:59Z.** Tudo o que as entradas 1.34, 1.35 e 1.36 descreviam como *"em `develop`, ainda não em
  produção"* **vale agora em produção**, e é por isso que esta entrada existe. São três comportamentos,
  todos na publicação de modelo (`POST .../model`):

  - **o documento é validado contra o metamodelo**, e a publicação responde `400` listando todas as
    violações de uma vez (1.34);
  - **a chave do agregado tem de ser `<boundedContext.name>.<type>`** — antes, chave divergente fazia o
    evento ser descartado sem log, sem métrica e sem DLQ (1.34);
  - **`roles` no nível do agregado** é aceito na publicação, e o recorte **declarado** é conferido, com
    cinco recusas em `400` (1.36).

  ⚠️ **Conferir a declaração não é aplicar o recorte.** O que subiu em produção é a publicação passar a
  aceitar e validar a chave. **Quem aplica o corte na leitura é o interpreter, e esse lado ainda não
  está em produção** (ver 1.38 — segue em `:amd64-260913c`). Até ele subir, declarar `roles` publica e
  **não recorta nada**: quem precisa do corte valendo espera a imagem do interpreter.

- **O que continua valendo, e é o que evita susto:** modelo **já publicado não é revalidado** — a
  checagem é na publicação. Quem publicou ontem e não publica hoje não muda de comportamento por causa
  desta subida. Agregado sem `roles` publica e lê como antes.

- **Como ler as datas de vigência daqui para trás:** as entradas 1.34, 1.35 e 1.36 dizem *"ainda não em
  produção"* porque era verdade quando foram escritas. **Esta entrada é o corte.** As páginas de uso
  descrevem o comportamento corrente e não carregam mais vigência; quem precisa saber *desde quando*
  vem a este arquivo.

## 1.36 — 2026-09-21

- **O aggregate passa a aceitar `roles`, que declara o recorte de LEITURA.** Opcional: quem não a usa
  publica e lê como antes. A forma é
  `{ read: [...], author: [...], scope: { read: { <PAPEL>: { rows: { by: <atributo> } } } } }`, e a
  publicação recusa com `400` o recorte incoerente — papel no `scope.read` fora de `roles.read`,
  `MASTER` no `scope`, `rows` sem `by`, `by` em metadado da plataforma, `by` em atributo não declarado
  em comando nenhum do aggregate. São as mesmas cinco recusas que o `accessControl.scope` da entity já
  fazia: o recorte é o mesmo dos dois lados, e divergir publicaria aggregate que passa e entity que não.

- **⚠️ `roles` agora existe em dois níveis, com formas diferentes.** No **comando** é **array** e
  **obrigatório**, e diz quem pode **executar**. No **aggregate** é **objeto** e **opcional**, e diz
  quem pode **ler**. O nível desambigua, e não há conversão entre os dois. Em
  [forger · model](forger/endpoints/model.md#recorte-de-leitura-do-aggregate-roles).

- Vigência: em `develop` do forger, **ainda não em produção**, como a 1.34 e a 1.35.

## 1.35 — 2026-09-21

- **Os exemplos de modelo deixaram de ensinar chave morta.** O introdutório
  [`acme.vendas.model.json`](examples/acme.vendas.model.json) perdeu `schema` e `concurrency` — nenhuma
  das duas é lida por serviço nenhum — e perdeu um `"length": 0` num atributo `Long`, onde `length` não
  significa nada. É o arquivo que a doc indica como ponto de partida, então era ele que estava
  ensinando o que não precisa existir. Os cinco "reais (sanitizados)" **continuam** com as duas chaves,
  porque espelham modelos publicados antes disso ser medido; o
  [README dos exemplos](examples/README.md) agora diz isso na cara, em vez de deixar quem copia
  descobrir sozinho.

- **`domainBus.triggerProjection` tem DUAS formas, e a doc só conhecia uma.** Item **string** é a
  projeção same-tenant; item **objeto** é a cross-tenant/cross-BC, e nela `targetTenantId` é
  **obrigatório** — sem ele o destino é descartado no despacho e a projeção não dispara. A publicação
  passa a recusar o objeto sem `targetTenantId`, e a recusa está na tabela de
  [forger · model](forger/endpoints/model.md).

- **Correção de rota:** a 1.34 tratava `triggerProjection` como lista de strings apenas. Estava errado
  desde que foi escrito, e quem denunciou foi o próprio metamodelo, ao reprovar um exemplo deste acervo
  que o motor aceita. A validação de publicação continua **em `develop` do forger, ainda não em
  produção** — vale a partir da imagem que a carregar.

## 1.34 — 2026-09-16

- **`POST .../model` passa a recusar com `400` modelo que o motor não executaria** — e a página
  [forger · model](forger/endpoints/model.md) descreve as três recusas mais comuns. ⚠️ **Em `develop` do
  forger, ainda NÃO em produção:** vale a partir da imagem que carregar essa mudança. Até lá a
  publicação continua aceitando os três casos, que seguem falhando — ou silenciando — em runtime, que é
  exatamente a razão da mudança.

  A mais cara das três: **a chave do agregado tem de ser `<boundedContext.name>.<type>`**. Ela é o
  endereço do agregado no despacho do evento *e* a chave que o cliente manda no comando; divergindo, o
  evento é descartado **sem log, sem métrica e sem DLQ** — do lado de fora, indistinguível de "nada
  aconteceu".

- **Chaves que um modelo novo não precisa trazer**, na mesma página: `schema.forWriteModel.name`,
  `schema.forReadModel.name`, `concurrency.*`, `readProjection` e `queue` **não são lidas por serviço
  nenhum**. Continuam aceitas — modelo publicado não precisa ser reescrito —, mas quem **gera** modelo
  novo não deve emiti-las. A página diz, para cada uma, quem decide de fato aquilo que ela aparenta
  decidir.

- **`boundedContext.name` documentado como o que ele é**: além de compor o endereço do agregado, ele
  **nomeia o schema** onde o log de consumo do evento é gravado. Divergir do dataschema do tenant faz o
  consumo gravar no schema de outro ambiente sem um único erro em log.

## 1.33 — 2026-09-16

- **Produção passou a rodar a mesma imagem do ambiente de teste: `yc-interpreter:amd64-260913c`.** Tudo
  o que as entradas 1.30, 1.31 e 1.32 descreviam como *"em teste; produção ainda não"* **vale agora
  também em produção**, e é por isso que esta entrada existe. São três comportamentos:

  - **datas normalizadas na entrada do comando**, com o agregado e a projeção devolvendo a mesma string,
    e o filtro de coluna `Timestamp` deixando de falhar (entradas 1.31 e 1.32);
  - **`NAO-MATERIALIZOU` como termo único** na busca de log de projeção que não materializou (1.31);
  - **consulta por campo dentro de atributo `Json`** valendo também quando o `valueObject` é *single*,
    e coluna vazia deixando de casar com `neq` (1.30).

- **Como ler as datas de vigência daqui para trás:** as entradas anteriores dizem *"produção ainda não"*
  porque era verdade quando foram escritas. **Esta entrada é o corte**: a partir de 2026-09-16, produção
  e teste rodam a mesma imagem para esses três itens. As páginas de uso descrevem o comportamento
  corrente e não carregam mais vigência — quem precisa saber *desde quando* vem a este arquivo.

- **Pré-requisito que acompanhou a promoção**, e que só interessa a quem opera banco: o event store de
  produção ganhou a coluna `client.aggregate.natural_key` e o índice único parcial que a acompanha, sem
  os quais essa imagem recusa a criação de agregado de modelo que declare `identity.fields`.

- ⚠️ **Mudança de comportamento que vem junto, e é visível para quem usa a API:** com a chave natural
  valendo, **o segundo agregado criado com a mesma combinação declarada em `identity.fields` é recusado
  pelo banco** — antes, dois iguais eram aceitos em silêncio. Vale para os agregados criados **a partir
  de agora**: os anteriores não foram verificados e podem conter repetições, como a entrada 1.28 já
  advertia.

## 1.32 — 2026-09-13

- **Datas são normalizadas na entrada do comando, e os dois endpoints passam a devolver a mesma string.**
  Antes, a data do cliente atravessava a plataforma crua: o endpoint de agregado a devolvia como veio — com
  fuso, com fração, com espaço — e o de projeção devolvia a forma do banco. O mesmo instante em duas
  grafias. Agora todo `Timestamp` sai como `2026-09-12T20:19:09` (UTC) e todo `Date` como `2026-09-13`, em
  [persistence-crs/spec/model-format.md](persistence-crs/spec/model-format.md).

- **Inclusive a data que a regra de negócio devolve.** A resposta do processador é mesclada depois de o
  comando ser lido, e passa pela mesma normalização.

- **`Date` não tem conversão de fuso:** vale o dia como escrito.

- ⚠️ **Mudança visível:** o carimbo automático (`whenAttribute`) no endpoint de agregado passa de
  `2026-09-12 20:19:09` para `2026-09-12T20:19:09` — igual ao que a projeção já devolvia.

- **A documentação de datas foi para onde cada leitor procura, com exemplo:** a regra e a tabela de
  formas aceitas em [spec/model-format.md](persistence-crs/spec/model-format.md); um corpo de comando com
  data em [endpoints/comando.md](persistence-crs/endpoints/comando.md); o estado do agregado com datas em
  [endpoints/agregado-leitura.md](persistence-crs/endpoints/agregado-leitura.md); e **como converter para
  o horário de Brasília no front** em
  [persistence-q/endpoints/consulta.md](persistence-q/endpoints/consulta.md#resposta) — com o erro de
  esquecer o `Z`, que desloca 3 horas em silêncio.

- **Vigência:** em teste desde `yc-interpreter:amd64-260913c` (`yc.cqrs-c@0a5c450`, `yc.dbr-driver@0334d08`).
  Produção ainda não.

- ⚠️ **A imagem anterior, `amd64-260913b`, trazia a normalização com um defeito:** a data com `T` era recusada
  pelo driver do banco na **criação** de registro — o comando respondia `200` e a projeção nunca
  aparecia. Transições funcionavam. Consertado em `amd64-260913c` e validado com dado real. Se você criou
  registro com data nesse intervalo e a linha não apareceu, **não recrie**: a mensagem está guardada e é
  reprocessável.

## 1.31 — 2026-09-13

- **Carimbos de tempo são gravados e devolvidos em UTC, sem fuso — agora escrito.** A conversão para o fuso
  de quem lê (`-03:00`, no Brasil) é do front. Não é convenção escolhida: é o que está gravado, medido nos
  serviços e nos bancos. Um `2026-09-12T20:19:09` devolvido significa 20:19 UTC, 17:19 em Brasília. Em
  [persistence-q/query-controls.md](persistence-q/query-controls.md#filtrar-por-data-e-hora-date-timestamp).

- **Filtrar por coluna `Timestamp` passa a funcionar.** Respondia `510` para praticamente qualquer valor:
  só um formato era aceito — com espaço **e fração obrigatória** —, e nem o valor que a própria consulta
  devolve podia ser mandado de volta. Passam a valer sem fuso (lido como UTC), só data, com `Z` ou
  `-03:00` (convertido para UTC) e epoch em milissegundos. Formato irreconhecível vira `400` com a lista
  do que vale.

- ⚠️ **O erro mais fácil de cometer está em destaque:** mandar hora local sem fuso. `"2026-09-13
  00:00:00"` é meia-noite **UTC** — 21:00 do dia anterior em Brasília. A seção traz o filtro de um dia
  inteiro em Brasília com o fuso junto.

- **Vigência:** em teste desde `yc-interpreter:amd64-260913b` (`ufrn.loco3@66652cf`) — o filtro funcionava nessa
  imagem; a criação com data, só a partir de `amd64-260913c`. Produção ainda não.
  Onde a imagem for anterior, só `"2026-09-13 00:00:00.000"`, com a fração e já em UTC.

## 1.30 — 2026-09-12

- **Dá para filtrar pelos campos de dentro de um atributo `Json`, e agora isso está escrito.** A
  capacidade existia e não constava em lugar nenhum da documentação: o critério aceita um nível a mais,
  com os campos internos do valueObject. Em
  [persistence-q/query-controls.md](persistence-q/query-controls.md#filtrar-por-campo-dentro-de-um-atributo-json).

- **As duas formas do valueObject passam a funcionar.** A escrita sempre aceitou objeto (`single`) e
  lista de objetos (`multiple`), e a leitura devolve as duas — mas o filtro só sabia lidar com lista, e
  **falhava** quando a coluna guardava um objeto. Quem modelou um valueObject `single` não conseguia
  consultar por dentro dele.

- **Passam a valer `neq`, `gt`, `gte`, `lt`, `lte` e `in`**, além de `eq`, `like` e `ilike`. Operador
  fora da lista continua sendo `400` que nomeia o operador — a lista cresceu, o silêncio não voltou.

- ⚠️ **O tipo do valor que você envia decide a comparação.** `{"gt": 10}` compara como número; `{"gt":
  "10"}` compara como texto, e aí `"9"` é maior que `"10"`. É a única informação de tipo disponível — o
  modelo declara a coluna como `Json` e não descreve o que existe dentro. **As aspas mudam o resultado.**

- **Coluna vazia não casa com nada, nem com `neq`** — e isso teve de ser consertado antes de sair.
  Ausência de valor respondia **duas coisas diferentes** conforme estivesse gravada como lista vazia ou
  como nula: a nula aparecia num filtro "diferente de". Agora as duas ficam de fora, igual.

- **Vigência:** em teste desde `yc-interpreter:amd64-260913` (`ufrn.loco3@6152345`). Produção ainda não.

## 1.29 — 2026-09-12

- **Passar `unique` de `false` para `true` deixou de ser recusado pelo forger — quem julga duplicata é
  o banco.** A recusa anterior dizia "requires all existing values to be unique" e **nunca consultava a
  tabela**: recusava também com a tabela **vazia**, que é o caso mais comum de quem está modelando. Pior,
  trancava o sistema contra si mesmo, porque a conferência de chave manda marcar o atributo como
  `unique` quando o agregado declara `identity.fields` de um campo — a instrução da recusa era proibida
  pela validação seguinte, e quem a seguisse não tinha saída. Vale para atributo e para associação.

- **A recusa da conferência de projeção passa a dizer COMO SAIR, e não só a causa.** O dataschema fica
  em `MODELING` de propósito, mas **fechar de novo dá a mesma recusa**: a única saída é retirar a
  declaração (`_conf.projectionOf: []`) e fechar. É o oposto do que quem está corrigindo tenta fazer,
  então ninguém descobre por tentativa — e até descobrir, o contexto fica inoperante. Em
  [forger/erros.md](forger/erros.md#projeção-que-não-confere-com-o-modelo-de-escrita--400-na-volta-a-running).

- **Vigência:** `forger@7e32720`, em teste desde `yc-composer:amd64-260912e`.

## 1.28 — 2026-09-12

- **A conferência de chave passou a olhar os DOIS lugares também quando `identity.fields` está vazio.**
  A versão anterior recusava `unique` num atributo sem lastro, mas deixava passar `_conf.uniqueKey` — é a
  mesma unicidade sem respaldo no modelo de escrita, só declarada no outro lugar, e é justamente a forma
  que uma chave composta usa. Bastaria um agregado esvaziar o `identity.fields` para a projeção seguir
  impondo a chave composta em silêncio, que é a classe de defeito que esta conferência existe para pegar.
  Quadro atualizado em
  [forger/endpoints/entity.md](forger/endpoints/entity.md#a-chave-identityfields-do-agregado-contra-a-da-projeção).
  **Vigência:** `forger@e95f65c`, em teste desde `yc-composer:amd64-260912d`.

## 1.27 — 2026-09-12

- **A conferência da projeção passa a cobrir também a CHAVE, e ela mora em dois lugares conforme a
  cardinalidade.** O `identity.fields` do agregado é o espaço de chave do modelo de escrita; na projeção,
  chave de **um** campo se declara no atributo (`unique: true`) e chave de **dois ou mais** em
  `_conf.uniqueKey`, que é o lugar da composta e exige 2+ atributos. A conferência segue essa divisão em
  vez de olhar sempre para o mesmo campo. Em
  [forger/endpoints/entity.md](forger/endpoints/entity.md#a-chave-identityfields-do-agregado-contra-a-da-projeção).

- **O caso que motivou:** um agregado deixou de exigir chave natural — `identity.fields` virou vazio —
  e a coluna continuou única na projeção. O segundo registro legítimo nunca materializava, sem erro
  visível a quem chamou. Agora `unique` sem lastro no `identity.fields` é recusado no fechamento da
  edição.

- **Dois limites ditos com todas as letras, porque calar seria pior:** com **dois ou mais** agregados em
  `projectionOf` a conferência de chave é **pulada** — cada agregado tem o seu `identity.fields`, e
  uni-los inventaria uma chave que nenhum declarou; e a conferência compara **definição com definição**,
  então **índice único já existente no banco** que tenha saído da definição **não é alcançado** — mudar
  o modelo não derruba índice criado antes.

- **Vigência:** `forger@eb82d28`, em teste desde `yc-composer:amd64-260912c`.

## 1.26 — 2026-09-12

- **A entity passa a poder declarar de que agregados ela é projeção — `_conf.projectionOf` — e o
  fechamento da edição confere as duas coisas contra o modelo de escrita.** Até aqui a tabela de leitura
  e a definição de escrita descreviam a mesma coisa, eram publicadas por caminhos diferentes, e **nada
  as confrontava**: a divergência só aparecia quando um evento tentava gravar, e aí não havia conserto —
  o comando já respondera `200`, a projeção falhava depois, e reemitir o comando falha com o agregado em
  outro estado. Em [forger/endpoints/entity.md](forger/endpoints/entity.md#declarar-que-a-entity-é-projeção--_confprojectionof-regra),
  com as recusas em [forger/erros.md](forger/erros.md#projeção-que-não-confere-com-o-modelo-de-escrita--400-na-volta-a-running).

- **São dois sentidos, e faltar é muito pior que sobrar.** *Teto*: a projeção não pode declarar atributo
  que agregado nenhum dos nomeados escreve — o efeito é coluna órfã, sempre vazia. *Piso*: a projeção
  tem de declarar **tudo** o que eles escrevem, carimbo de evento incluído — sem a coluna, o motor
  recusa a gravação **inteira** e a linha nunca materializa. Por isso **a projeção não pode ser mais
  estreita que o agregado**: não é escolha de desenho, é que uma chave ausente derruba a gravação toda,
  não só aquele campo.

- **A conferência roda no fechamento da edição, e não na publicação de cada artefato.** A entity e o
  `.model.json` são publicados por caminhos diferentes e em **qualquer ordem**; conferir um contra o
  outro no ato de publicar recusaria sequência legítima, porque o segundo artefato ainda não existe. Na
  transição `MODELING → RUNNING` os dois estão na mesa. A recusa é `400`, **nada é gravado**, e o
  dataschema **continua em `MODELING`**.

- **O item de `projectionOf` é o `type` do agregado, não a chave do mapa `aggregate`.** No `.model.json`
  o agregado aparece sob chave composta (`"ensino.aula"`) e tem um campo `"type": "aula"`; o roteamento
  da projeção usa o `type` cru. A recusa lista os **dois** vocabulários, porque o erro provável é copiar
  o que se vê no arquivo.

- **É opt-in, e isso é o que protege quem já existe:** entity sem `projectionOf` não é conferida, e
  nenhum dataschema publicado hoje passa a ser recusado. **Vigência:** a partir da imagem que carregar
  `forger@04ee971` — em teste desde `yc-composer:amd64-260912`.

- **Corrigido: recurso inexistente respondia `510 "Falha inesperada, solicite suporte."` em vez de
  `404`.** Valia para entity, dataschema, atributo e associação não encontrados — oito lançamentos do
  mesmo controller, todos sem o prefixo de status que o tradutor de erro usa para decidir o código. Quem
  integra não conseguia distinguir "não existe" de "o serviço quebrou", e `510` manda procurar suporte
  por uma condição perfeitamente normal. O catálogo já documentava `404`; era o comportamento que estava
  errado. **Vigência:** `forger@5c13391`, em teste desde
  `yc-composer:amd64-260912b`.

## 1.25 — 2026-09-10

- **O que separa teste de produção são dois critérios, não um** — e o documento não dizia nenhum dos
  dois. [Arquitetura](01-arquitetura.md) ganha a seção *Teste e produção — o que separa um do outro*, na
  Fase 3: a **escrita** se separa pela **superfície** (o prefixo `/t/` entra numa instância de teste do
  motor, que escreve no event store de teste); a **leitura** se separa pelo **tenant**, e não por nome
  ou instância de banco — dois tenants podem ter o mesmo modelo de leitura, um de teste e outro de
  produção, materializados até no mesmo banco.

- **Por que a lacuna importava:** sem esse critério escrito, é natural inferir o ambiente do lugar onde
  o dado está — "esta linha está no banco X, logo é produção". A inferência é inválida, e leva a tratar
  como delicado um dado de teste, ou o contrário. A pergunta que classifica é **de que tenant é a
  linha**; o tenant de teste existe desde o começo, o de produção só quando o sistema vai ao ar.

## 1.25 — 2026-09-12

- **"O comando respondeu `200` e a linha não apareceu" ganhou seção própria, com a receita para achar a
  causa.** É o sintoma mais caro da plataforma, e o mal-entendido está no `200`: ele responde pela
  **escrita**. A projeção é aplicada depois, a partir de uma fila, e quando falha não há para quem
  responder — quem chamou já foi embora. A seção diz o que fazer nessa ordem: conferir a projeção depois
  de qualquer transição nova, procurar o rastro no log de serviço, e ler o que voltou. Em
  [persistence-crs/erros.md](persistence-crs/erros.md#o-comando-respondeu-200-e-a-linha-não-apareceu),
  com ponteiro do [persistence-q](persistence-q/README.md).

- **A busca virou uma palavra só: `NAO-MATERIALIZOU`.** A mesma classe de falha saía sob três termos
  diferentes, conforme o caminho — projeção do tenant, projeção entre contextos, saga —, e o endpoint de
  logs recebe o termo **na própria rota**. Quem errava a palavra recebia `204`, que é indistinguível de
  "não houve falha": foi assim que um cliente concluiu que a falha era silenciosa quando ela estava
  registrada sob outro nome. **Vigência:** em teste desde `yc-interpreter:amd64-260913`
  (`persistence-crs@9bc1b28`); em produção, os três termos antigos continuam sendo o caminho.

- **Registrado também que `204` na busca de log significa "não achei esse termo"** — nunca "não houve
  falha". A distinção não era óbvia e custou uma investigação inteira.

## 1.24 — 2026-09-10

- **Pare de invalidar cache à mão: a plataforma já repõe o modelo sozinha.** Até esta versão, a
  documentação do [persistence-crs](persistence-crs/README.md#depois-de-alterar-o-modelo) e do
  [persistence-q](persistence-q/README.md) mandava, depois de alterar um modelo, **apagar duas chaves de
  cache** antes de republicar. Era **trabalho desnecessário**: fechar a edição do dataschema
  (`MODELING` → `RUNNING`) já republica o modelo de leitura, e republicar o `.model.json` já sobrescreve
  o de escrita. Quem seguia aquele texto fazia uma operação delicada, à mão, sem precisar. **Se é o seu
  caso, pode parar.**

- **E são dois modelos, não duas cópias do mesmo** — o que a redação anterior escondia. O **de escrita**
  (agregados e comandos) e o **de leitura** (projeções) têm gatilhos diferentes e não se atualizam
  juntos. A seção agora diz, numa tabela, o que repõe cada um.

- **O que morde não é editar entities: é deixar a edição aberta.** Enquanto o dataschema está em
  `MODELING` o modelo de leitura não existe, e a consulta falha com `510`. Ele volta ao fechar. Isso não
  estava dito em lugar nenhum, e a ausência fazia o `510` parecer defeito.

- **A edição de um schema passa a fechar o tenant inteiro.** Até aqui, entrar em edição fazia a consulta
  falhar e **o comando continuar sendo aceito** sobre um modelo em remodelagem. Agora a janela recusa os
  dois. O mecanismo, o roteiro e o pré-requisito para voltar a `RUNNING` estão em
  [forger/dataschema](forger/endpoints/dataschema.md#o-que-a-transição-faz-com-o-modelo-no-cache), que é
  a fatia de quem publica e remove os modelos.

- **A cópia em memória de cada instância deixou de existir na instância de teste.** Até 2026-09-10 cada
  instância guardava o modelo por até uma hora, e uma alteração podia "pegar" numa e não noutra. No
  `tinterpreter`, desde a imagem `amd64-260910`, essas cópias estão **desligadas**: o modelo é relido a
  cada requisição e **republicar passa a valer na requisição seguinte**. Onde continuarem ligadas, a
  defasagem de até uma hora ainda vale.

## 1.23 — 2026-09-09

- **Uma entity pode declarar que um papel lê apenas as linhas de que o usuário é titular.** Até aqui o
  controle de acesso só sabia dizer *"este papel lê esta tabela"* ou *"não lê"* — não havia como dizer
  *"lê só o que é dele"*, e a consequência era que qualquer usuário final autenticado lia a tabela
  inteira de dados pessoais dos demais. A declaração é `accessControl.scope`, e as duas metades estão
  documentadas onde cada uma acontece: **como declarar** em [forger — entity](forger/endpoints/entity.md)
  e **o efeito na consulta** em [persistence-q](persistence-q/README.md#recorte-de-leitura-por-titular).

- **A seção do recorte deixou de ser lista de consequências e virou percurso.** Ela respondia *"o que
  acontece se…"* sete vezes e nunca *"como eu faço"*. Agora vai do modelo declarado à resposta, com a
  requisição no meio — e ganhou o que faltava para modelar sem errar: **o que torna uma linha "sua"**, a
  tabela-verdade de `read` × `scope`, e **como o processador deve ser escrito**.

- **`scope` nunca concede acesso, só estreita o que o `read` já permitiu.** Era a regra que amarrava
  tudo e não estava escrita em lugar nenhum: papel fora do `read` não lê e portanto nunca chega ao
  recorte; papel no `read` e fora do `scope` lê a tabela inteira. Sem ela, *"acumular papéis amplia"*
  soava arbitrário em vez de decorrer da regra.

- **⚠️ Atributo de titular vazio torna a ficha invisível para o próprio dono** — com `200` e lista
  vazia, indistinguível de *"não existe"*. É a armadilha de modelagem mais provável do recurso, e ela
  vale especialmente quando a ficha é cadastrada por outra pessoa. Está em
  [o que torna uma linha "sua"](persistence-q/README.md#o-que-torna-uma-linha-sua).

- **Usar o token do usuário num processador que precisa ver dado de terceiro devolve lista vazia com
  `200`** — não um erro. Quando a entity é recortada, aquele token restringe a leitura ao próprio
  usuário. Para ver além, o caminho é o endpoint interno de cluster.

- **Errata em `identity.fields`: a garantia prometida não existia, e agora existe.** O texto afirmava
  que a combinação declarada era única e que *"não pode haver dois registros com a mesma combinação"* —
  **nada no processamento do comando lia esse campo**, e dois agregados iguais eram aceitos. A partir de
  `amd64-260909c` a combinação **é imposta na criação**. Consequência para quem já tem dados: **os
  registros criados antes não foram verificados** e podem conter repetições. Ver
  [identidade e unicidade](persistence-crs/spec/model-format.md#identidade-e-unicidade-identity).

- **E como escolher os `fields`, que faltava.** A pergunta não é *"quais campos são obrigatórios"*, é
  **"o que faz dois destes serem o mesmo?"**. Com a armadilha por extenso: **recriar depois de encerrar
  costuma ser legítimo** — o aluno que sai da turma e volta no semestre seguinte é matrícula nova, e uma
  chave que não os distingue passaria a recusar operação correta.

- **Toda afirmação sensível a versão passou a ser datada por versão, e não por expectativa.** Duas vezes
  no mesmo dia um texto envelheceu em horas: uma ressalva de indisponibilidade que a implantação
  desmentiu, e uma errata que a implementação desmentiu. Doc ausente faz perguntar; **doc errada faz
  agir**.

## 1.22 — 2026-09-09

- **O br-service tem TRÊS contratos, não um — e a doc apresentava um só.** O serviço é acionado de três
  lugares (regra de negócio no caminho do comando, coordenação por fila, projeção cross-contexto por
  fila) e os três mandam **corpo diferente**, entregam **número diferente de argumentos** ao processador
  e esperam **forma de resposta diferente**. A fatia ganha um documento que é o eixo disso,
  [br-service — os três contextos de invocação](br-service/contextos.md), e o guia passa a ser porta de
  entrada por pergunta.

- **O envelope `processedData` é obrigatório nos dois contextos assíncronos e proibido no síncrono.**
  Esta é a correção mais cara da versão: o guia documentava só a forma **sem** envelope e os exemplos só
  a forma **com** envelope — cada um certo sobre um contexto e errado sobre o outro, e nenhum dizia de
  qual falava. Um processador de coordenação que devolve o objeto direto recebe `200` e o comando-alvo
  **nunca nasce**, sem erro em lugar nenhum.

- **Não existe fan-out de coordenação.** Uma coordenação emite **um** comando, nunca uma lista — e
  `targetCommand` ausente faz a saga ser descartada em silêncio, com a mensagem confirmada. Quem
  esperava encerrar N filhos a partir de um cancelamento não tinha como saber que o mecanismo não
  existe.

- **Na projeção, `aggregateid` é obrigatório** no objeto devolvido, e a chave tem de ser o nome da
  **projeção de destino** — é o que decide entre criar e atualizar a linha.

- **`accessControl` visto de dentro do processador ganha documento próprio**
  ([br-service — acesso a dados](br-service/acesso-a-dados.md)): com que identidade ler, por que usar o
  token do usuário para validar contra dado de **terceiro** devolve **lista vazia com `200`** — e não um
  erro —, quais controles de acesso mordem em que momento, e o diagnóstico reproduzível para o sintoma
  "comando `200`, processador ok, **projeção não aparece**". Vale a partir de `amd64-260909b`; a
  ressalva de que o controle de escrita está no artefato e **não foi exercitado** está escrita lá.

- **O `400` do br-service não é o que o cliente final vê.** No contexto síncrono, a falha do processador
  chega ao cliente como **`510`**. Quem tratava `400` nunca encontrava a mensagem do processador.

- **`authToken` e `tenantIds` passam a constar do contrato de fio** ([executar função](br-service/endpoints/br.md)
  e o OpenAPI da fatia): são eles que decidem **quantos argumentos** o processador recebe, e estavam
  ausentes do corpo publicado. E o `authToken` **nunca** existe em contexto assíncrono — não é "pode não
  haver".

## 1.21 — 2026-09-06

- **Errata na doc da borda: cada verificação assina com um cabeçalho próprio.** A versão 1.20 dizia
  que resposta da borda se reconhece por `X-Blocked-By`. **É falso para tamanho e para vazão:** um
  `413` da borda traz `X-Size-Limit-Exceeded` (com `X-Max-Size-Allowed` e `X-Actual-Size`), e um `429`
  traz `X-RateLimit-…`. Quem procurasse `X-Blocked-By` num `413` não o acharia e concluiria que a
  resposta veio do servidor de entrada — errando exatamente no discriminador que a página existe para
  dar.

- **Insistir depois do `429` muda o código para `403`.** Origem que continua chamando é banida por
  **7 dias** e passa a receber `403` com `X-RateLimit-…`, não `429`. Cliente que só trata `429` vê o
  erro mudar de natureza sem explicação.

- **A ordem das verificações não é a intuitiva, e muda o erro que você recebe.** A checagem de rota é
  das **últimas**, não das primeiras: caminho inexistente **com corpo grande** recebe `413`, não
  `404` — e como caminho desconhecido cai no teto mais restritivo, o `413` aparece com folga. Quem
  receber `413` em rota nova deve conferir **se a rota está publicada** antes de olhar o payload.

- **`403` tem quatro origens distintas**, e os cabeçalhos as separam — inclusive uma que **não é da
  borda**: `403` sem nenhum cabeçalho `X-` vem do servidor de entrada, que recusa caminho sem rota
  publicada antes de a requisição chegar à borda. Correção diferente, lugar diferente.

- **Nem todo serviço fica atrás da borda.** O de arquivos estáticos responde por domínio próprio.
  Ausência de cabeçalho da borda ali **não** significa que ela deixou passar — significa que ela
  nunca viu a requisição.

- **A isenção de cabeçalho para canal persistente vale para caminhos nomeados**, não para "qualquer
  canal": a 1.20 generalizava, e abrir canal em outra rota é recusado como qualquer requisição.

## 1.20 — 2026-09-06

- **A borda ganha documentação própria, e ela responde a uma pergunta que hoje não tem resposta:
  quem produziu este erro?** Uma recusa da borda e uma recusa do serviço chegam pelo mesmo canal e,
  em vários casos, com o **mesmo código HTTP** — e quem investiga procura o defeito no serviço que
  nunca foi chamado. O novo [catálogo de erros da borda](gateway/erros.md) dá o discriminador: os
  cabeçalhos `X-Blocked-By` e `X-Blocked-Reason` só existem em resposta gerada pela borda. `404` com
  eles é rota não reconhecida antes do encaminhamento; `404` sem eles é recurso inexistente dentro do
  serviço. Vale igual para o `413`, que pode vir da borda ou do proxy à frente dela.

- **A regra de um só cabeçalho de identificação passa a estar escrita.** Toda requisição carrega
  `X-Tenant-Id` **ou** `X-Forger-Credential` — **nunca os dois, nunca nenhum** —, e qual dos dois
  depende da rota: execução exige o tenant, administração exige a credencial, autenticação aceita
  qualquer um. Mandar os dois não é mais permissivo: é `400`. Era o erro mais comum contra a borda e
  só estava documentado de lado, dentro da página de outro serviço.

- **Errata de comportamento:** desde 2026-09-04 a borda **põe o motivo no corpo** quando a recusa é
  erro de quem chamou (cabeçalho ausente, cabeçalhos conflitantes, formato inválido). Detecção de
  ataque continua opaca de propósito. Descrições de que a borda responde sempre com corpo vazio
  estão desatualizadas.

- **Escopo declarado:** a borda **não tem endpoint próprio exposto ao cliente**, então não há
  `endpoints/` — os endpoints administrativos dela são restritos por origem e ficam fora da
  convenção, que cobre apenas endpoints autenticados e expostos ao cliente. Segue o precedente de
  `cache`, componente sem rota pública que também é documentado por guia.

## 1.19 — 2026-09-05

- **A resposta passa a dizer quando veio cortada.** Toda consulta tem teto — o `_maxRegisters` que você
  mandou, ou o padrão (**500** no modo array, **1000** no modo objeto). Até aqui o corte era invisível:
  uma lista de 500 itens era indistinguível de uma tabela com 500 registros, e a tela mostrava uma parte
  acreditando ser o todo. Agora, **quando há mais além do teto**, o item da resposta traz
  `_truncated: true` e `_maxRegisters`. A marca só aparece quando sobrou algo — **a ausência dela é a
  garantia de que a lista está inteira**, inclusive quando ela tem exatamente o tamanho do teto. Ao
  percorrer as chaves do item, pule as que começam com `_`: o rótulo da entidade é a que não começa.

- **`_count` devolve `totalRegisters` — o campo que a doc prometia e o código nunca escrevia.** A
  contagem vinha como número solto sob o rótulo, e a constante do nome existia sem nenhum uso: quem
  seguia a doc procurava um campo que não chegava, sem erro para denunciar. Agora a resposta é
  `{"<rótulo>": {"totalRegisters": N}}`, respeitando o mesmo filtro da consulta. É o caminho para saber
  quantos existem antes de decidir como paginar.

- **`204` volta a existir no modo objeto.** A decisão usava o número de **consultas** do lote, que é
  sempre ≥ 1 — então resultado vazio saía como `200` com `[{"<rótulo>": []}]`, contrariando a garantia
  publicada. O modo array já fazia a verificação certa; agora os dois fazem.

- **A seção "Garantias da resposta" dizia "registros encontrados", e isso afirmava completude.** Sob
  truncamento a frase era falsa, e a página do endpoint sequer mencionava a existência de um teto — o
  aviso morava só em `query-controls.md`, e o ponteiro daqui apontava para o vazio. A garantia foi
  reescrita, o teto e a marca de corte entraram na página do endpoint, e a descrição do `200` no
  contrato de máquina (`openapi.yaml`) deixou de omiti-los.

- **`_associations`, `_populating`, `_level` e `_as` saíram da doc: nenhum deles é lido pelo código.**
  A página os descrevia como o jeito de popular relacionados, inclusive com "profundidade de
  população" — os quatro eram inertes, e como toda chave `_` é isenta da validação de vocabulário, nem
  `400` havia. No lugar entrou o mecanismo real: a associação vem junto quando entra no critério, como
  objeto aninhado sob o nome dela.

- **Critério de associação com mais de um item passa a ser `400`.** Só o primeiro era visitado; os
  demais eram descartados em silêncio e a resposta parecia completa. Enquanto atender vários não for
  possível, a recusa é explícita — "envie uma consulta por item".

## 1.18 — 2026-09-05

- **Os controles de consulta são irmãos do rótulo, não filhos dele — a doc dizia o contrário.** A página
  mandava pôr `_paging`, `_sorting`, `_count`, `_connective` e `_cache` **dentro** do objeto do rótulo;
  o motor os lê no **mesmo nível** do rótulo. Quem seguiu a doc não recebeu erro: o controle era
  **ignorado em silêncio** e a consulta rodava com o padrão — página de 50 virava a página padrão,
  `OR` virava `AND`. A validação de nome de atributo não pega isso, porque isenta toda chave `_`. É a
  correção mais importante desta revisão: qualquer cliente escrito pela página anterior está com os
  controles inertes. Atualizados [`persistence-q/query-controls.md`](persistence-q/query-controls.md) e
  [`persistence-q/endpoints/consulta.md`](persistence-q/endpoints/consulta.md).

- **Controle que a plataforma não vai honrar passa a ser `400` na entrada.** Sete situações que antes
  saíam como `510` de origem obscura, ou — pior — rodavam devolvendo coisa diferente da pedida:
  `_connective` fora de `AND`/`OR` maiúsculo corrompia o filtro; `_paging` pela metade (inclusive `{}`)
  produzia consulta **sem limite nenhum**, trazendo a tabela inteira; `_sorting` na forma plana era
  aceito e a consulta saía **sem ordenação**, sem aviso; `_order` fora de `ASC`/`DESC` e `_orderBy` fora
  do modelo iam crus para dentro da ordenação; `in` com lista vazia estourava; operador diferente de
  `eq`/`like`/`ilike` num atributo `Json` virava **igualdade em silêncio** — quem pedia `gt` recebia o
  resultado de `=`. E `_count` numa consulta simples produzia consulta inválida. Todos recusados ou
  corrigidos, com mensagem que nomeia o controle e diz a forma esperada.

- **O que o motor sempre fez e a página nunca contou.** Passa a estar documentado: o operador
  `distinct`; o limite padrão quando `_paging` é omitido (**500** no modo array, **1000** no modo
  objeto) — omitir não traz tudo; o `%` **obrigatório** em `like`/`ilike` e a restrição a atributo
  textual; o `%` num valor simples virando busca textual **sem você pedir**; a forma indexada do
  `_sorting` e o fato de só `"0"`, `"1"` e `"2"` serem lidos; entidade particionada **ignorando** o seu
  `_orderBy`; a **faixa** formada por dois operadores no mesmo atributo (só `gt`/`gte` com `lt`/`lte`) e
  a chave `CONNECTIVE` em maiúsculas que inverte a junção dela; o `_connective` ser **global** e
  propagar para associações e componentes; valor vazio não virando filtro **nem coluna** no resultado; o
  `_ttl` obrigatório com `_cache: use`; `_cache` sem `_behavior` sendo ignorado e comportamento
  desconhecido sendo `400`.

- **`_cache`: a chave inclui os controles que você mandou, e o `evict` voltou a funcionar.** A chave da
  entrada é formada **antes** de `_paging`, `_sorting`, `_connective` e `_count` serem retirados — então
  a mesma consulta gravada **com** `_paging` explícito e relida **sem** ele produz chaves diferentes, e o
  resultado não é reaproveitado. O `evict` deixou de ser inócuo: ele e o `use` passaram a formar a chave
  do mesmo jeito, então a invalidação alcança o que foi gravado (desde que mandada com os mesmos
  controles). O aviso sobre o `use` foi reescrito para descrever o que de fato acontece: na **falta** a
  resposta vem **sem os registros e a consulta sequer é executada** (o serviço não distingue "não achei"
  de "achei"); no **acerto** o cliente recebe o invólucro da entrada; e no **modo objeto** ainda sobra um
  segundo item, porque a consulta roda mesmo assim.

- **O rótulo é o nome exato da projeção.** Os exemplos usavam o plural (`"pedidos"`), que só funciona se
  a projeção tiver sido modelada com esse nome. Corrigidos para o nome no singular, e a regra ficou
  escrita: rótulo fora do modelo é `400`, com a lista dos nomes declarados na mensagem.

- **O estado `RUNNING` é conferido uma vez por tenant em cada instância, não a cada consulta.** A página
  apresentava a checagem como se valesse por requisição. Não vale: depois da primeira requisição daquele
  tenant, a instância continua atendendo sem reavaliar o estado. Então **voltar o modelo para edição não
  interrompe** quem já está sendo atendido, e a mesma mudança pode "pegar" numa instância e não noutra.
  Atualizado [`persistence-q/README.md`](persistence-q/README.md).

## 1.17 — 2026-09-05

- **Atributo fora do modelo agora é `400` — antes a consulta devolvia tudo.** A página de consulta já
  dizia que nome desconhecido dava erro; não dava. O motor percorre o vocabulário da entity e pergunta se
  o pedido tem aquilo, então o nome que não existe nunca era visitado: o filtro sumia e a consulta rodava
  sem ele. Quem escrevia `cpff` por engano não recebia erro nem lista vazia — recebia **todos** os
  registros, acreditando ter filtrado. Agora os nomes desconhecidos são recusados juntos, numa só
  mensagem, com a lista dos declarados. Chaves de controle (`_`-prefixadas) seguem livres.

- **Rota nova: `POST /e/by` — remoção por predicado.** Até aqui o `/e` só apagava pela chave primária:
  quem mandasse qualquer outro critério recebia `200` e **nenhuma linha era removida**, em silêncio,
  porque o montador acrescentava uma condição sobre `id` que nunca casava. A rota nova apaga pelo critério
  informado, e por apagar **N linhas de uma vez** tem contrato próprio: `where` vazio é recusado,
  `_connective` é explícito (no `/e` o `AND` implícito inverte o resultado de quem esperava `OR`),
  `_maxRows` é obrigatório e a contagem acontece **antes** — se o filtro casar mais que o teto, a resposta
  é `409` e nada é apagado —, e `_dryRun` é `true` por padrão, então a primeira chamada mostra quantas
  linhas casariam sem tocar em nada. Nova página
  [`persistence-crs/endpoints/remocao-por-predicado.md`](persistence-crs/endpoints/remocao-por-predicado.md),
  indexada no `llms.txt` e no guia do serviço.

- **`valueObject`: duas formas, e só essas duas — e escalar passa a ser `400`.** `single` é **um
  objeto**; `multiple` é **um array de objetos**. Vale para as duas maneiras de declarar (grupo de campos
  e atributo tipado direto): a forma do valor é a mesma nas duas. **Escalar nunca** — nem solto, nem
  dentro de array —, então `"diassemana": ["terca"]` deixa de ser aceito. O que existe **dentro** do
  objeto continua sendo seu: um campo ou dez, com objetos aninhados se o domínio pedir; a plataforma
  cobra a forma, não o conteúdo, e o aninhamento chega intacto ao agregado e volta intacto na consulta.
  Num `multiple`, **um único item fora da forma reprova o comando inteiro** — não há aproveitamento
  parcial. Antes a escrita reduzia tudo a escalar e a leitura tentava ler toda coluna `Json` como lista;
  as duas pontas discordavam, e o encontro delas era a atualização da projeção — o primeiro comando
  gravava e o segundo quebrava. Atualizado
  `persistence-crs/spec/model-format.md`.

- **Consulta de logs: virou `POST`, o segredo vai no corpo, e ele é conferido contra o tenant.** Três
  mudanças no mesmo endpoint. (1) O que autoriza é o **segredo do seu tenant**, e ele é conferido **contra
  o `X-Tenant-Id`**: apresentar o segredo de um tenant pedindo o log de outro é `403` — antes, quem tivesse
  a chave lia o log de qualquer cliente trocando o cabeçalho. (2) O segredo saiu do cabeçalho e foi para o
  **corpo**, e por isso o verbo passou a ser `POST` — em caminho de URL o segredo apareceria em registro de
  acesso e em histórico de cliente. A operação continua sendo leitura. Corpo sem `secret` é `400`, não
  `403`. (3) A URL publicada omitia o segmento `c`: o correto é `/v3/persistence/c/logs/...` (produção) e
  `/v3/persistence/t/c/logs/...` (teste) — sem o `c` a borda devolve `404`. Entram também o `500` na tabela
  de erros e a correção sobre o caminho de fila, que **aparece** na consulta.

- **es-n: a garantia de entrega vale para a projeção do próprio contexto, não para os três destinos.**
  Nos despachos de `triggerProjection` e `triggerCoordination` a falha de publicação não segura o
  checkpoint e não há reentrega automática. E o envelope leva **os dados do evento**, não o estado atual
  do agregado — o es-n não reconstitui nada. Atualizado `es-n/README.md`.

## 1.16 — 2026-09-05

- **Escrita que não acha o alvo responde `404`, não `204` — 31 operações do orgid.** A 1.15 corrigiu um
  caso (`PUT /ua/role`); a varredura mostrou que o padrão era idiomático no serviço. `204` é família 2xx —
  sucesso sem corpo —, então uma escrita que falhava era **indistinguível de uma que gravou**: os dois
  respondiam 2xx e vazio, `response.ok` era `true` nos dois, e a mensagem de erro era descartada pelo
  próprio protocolo, que proíbe corpo em `204`. Agora cada `404` traz **o efeito que não ocorreu**, não só
  o objeto que faltou: `"account not found: the password was not changed."`, `"O papel informado não
  existe: nada foi removido."` Os dois casos mais perigosos que isso encerra: `DELETE` de papel inexistente
  (num `DELETE`, `204` **é** o código canônico de sucesso — a leitura era a oposta da correta) e
  `replace-master-account`, em que trocar a conta administradora de uma organização virava no-op
  silencioso. **`204` continua em leitura**, incluindo os `/exists`, onde `200`/`204` é o contrato e não um
  erro. Atualizados `orgid/erros.md` (linha `404` nova + nota), `orgid/README.md` (ciclo de vida) e as
  seções de escrita de `ua-papel.md`, `ua-conta.md`, `ua-associacao.md`, `up-conta.md`,
  `up-organizacao.md`, `up-associacao.md` e `publico.md`.

- **⚠️ `/ua/open/role/owner/{owner}` é o único público do orgid fora do prefixo `/open/`.** Os segmentos
  estão invertidos, e por isso ele escapa de qualquer regra de autorização derivada de `/open/**` — no
  `OrgIdSecurityConfig` do orgid standalone ele **exige token apesar de documentado como público**. O
  `composer`, que é o que está implantado, libera os dois prefixos desde 2026-09-05. Registrado em
  `orgid/endpoints/publico.md` e `ua-papel.md`.

- **Duas correções de comportamento que a doc não trazia:** conta `/ua` nasce `PENDING` e **já loga** — o
  sign-in não checa `status`, e a ativação por hash serve para provar posse do e-mail, não como portão de
  acesso (`ua-conta.md`); e **registro não envia e-mail** — quem envia é o fluxo de hash e
  `/open/ua/account/e-mail/send`, contra provedor real com cota, o que desaconselha acioná-los em teste
  (`publico.md`).

## 1.15 — 2026-09-05

- **`PUT /ua/role` exige `id`, e a documentação não dizia.** O texto era "mesmos campos do POST" — e o
  POST **não tem `id`**. Quem seguia a doc ao pé da letra montava um corpo sem `id`, e o serviço
  respondia **`204`**: família 2xx, sucesso sem corpo. A mensagem "role not found." era descartada pelo
  próprio protocolo, e o chamador seguia achando que tinha configurado. Um cliente marcou dois papéis
  como públicos assim e perdeu tempo até descobrir que o valor no banco nunca mudara. A seção agora
  mostra o corpo com `id`, diz como descobri-lo (`GET /ua/role/by/name/{roleName}/owner/{roleOwner}`),
  e registra os códigos corretos: `400` sem `id`, `404` quando o papel não existe ou é de outro `owner`.
  O `204` foi corrigido no serviço na mesma data. Em `orgid/endpoints/ua-papel.md`.

## 1.14 — 2026-09-03

- **Correção de fato: não existe recuperação automática do modelo a partir do forger.** A documentação
  descrevia um *self-heal* — "se falta no cache, recupera do Forger e repõe" — que **não existe mais**: o
  serviço de implantação passou a ser o **único** publicador do modelo no cache, e o caminho de leitura
  **só lê**. Um miss é **parada** (`510`, "republique o modelo"), não um caminho lento que se resolve
  sozinho; e, como a entrada **não expira**, um miss significa "nunca publicado" ou "removido". Corrigido
  em `persistence-crs/README.md` (dono do fato) e `persistence-q/README.md`; consequência registrada em
  `03-fluxo-de-deploy.md` (a publicação é a única porta de entrada).
- **Regra nova: depois de um comando, espere antes de consultar.** O polling contra o serviço de consulta
  deve começar por um **atraso inicial em milissegundos** e reconsultar com tentativas limitadas — nunca
  disparar a primeira consulta junto com a resposta do comando. A razão passa a estar escrita: o
  mecanismo que persiste a **projeção** é **desvinculado** do que persiste o **agregado**, logo o `200`
  do comando **não afirma nada** sobre a projeção. Inclui a consequência que mais engana: uma falha na
  projeção **não aparece** na resposta do comando. Em `persistence-q/README.md`.
- **`_cache: "use"` marcado como não utilizável.** O controle está documentado, mas hoje erra nos dois
  caminhos: sem entrada, responde sem os registros **e a consulta nem executa**; com entrada, devolve o
  invólucro em vez do resultado. Enquanto não houver correção, a orientação é omitir `_cache` ou usar
  `"ignore"`. Em `persistence-q/query-controls.md`.
- **`envtype` é rótulo, não seletor.** O campo é obrigatório na criação do project, mas **não** escolhe
  banco, esquema, instância nem rota, e nenhum processamento o consulta — preenchê-lo com `PRODUCTION`
  não torna nada produtivo. Documentado como anotação administrativa, com ponteiro para a única forma
  de saber o destino real. Em `forger/endpoints/project.md`.
- **Nova seção: "descobrir para onde um tenant projeta".** O destino das projeções não está no modelo
  nem no `tenant-id`; vem do provisionamento. Documentado o percurso da cadeia
  `model → dataschema → database → dbconn` com os endpoints autenticados de cada passo, e o motivo:
  antes de qualquer ação com consequência sobre os dados de um tenant, é a cadeia que diz qual banco a
  ação alcança. Em `forger/README.md`.
- **Alterar `dbconn` não vale de imediato.** A conexão resolvida do tenant fica guardada em memória por
  um intervalo; até vencer, as requisições seguem usando endereço e credencial antigos — e uma falha de
  autenticação logo após a troca não significa que o novo valor está errado. Em
  `forger/endpoints/dbconn.md`.
- **Dois diagramas novos.** (1) *Como o `tenant-id` vira o banco de um cliente* — separa as duas
  resoluções que partem do mesmo `tenant-id`, a **conexão** (vem do forger) e o **modelo** (vem do
  cache), em `02-conceitos.md`. (2) *Qual banco de leitura, e quando isso é decidido* — mostra que o
  destino da projeção é resolvido **no consumo da fila**, a partir do `tenant-id` **da mensagem**, e
  portanto **depois** da resposta do comando, em `01-arquitetura.md` §Fase 3.

## 1.13 — 2026-09-02

- **`PUT` de entity: atualização parcial de verdade — ausente ≠ vazio.** A documentação já prometia
  *"campos omitidos são ignorados, não apagados"*, mas o serviço tratava propriedade **ausente** como
  pedido de reset ao default: um `PUT` que enviava só `accessControl` **apagava o `uniqueKey`** da
  entity, sem nada na resposta indicando a remoção. O mesmo mecanismo atingia `superEntity` (desfazia
  herança) e, no nível de atributo, derrubava `NOT NULL`/`unique` e apagava o comentário. Corrigido no
  serviço; o contrato agora é explícito:
  - propriedade **ausente** → ignorada (valor atual preservado);
  - propriedade com **valor vazio** (`[]`, `""`) → **limpa**;
  - `null` explícito conta como **ausente** (não é sentinela de limpeza);
  - `accessControl` é **substituído por inteiro**, não mesclado chave a chave (enviar só `read` reseta
    `write` para `["MASTER"]`);
  - `_conf` é **opcional** no `PUT` (segue obrigatório na criação).
- **`PUT` passou a responder com o que aplicou.** O `200` antes vinha **sem corpo**; agora traz
  `{entityName, valid, summary, totalChanges, applied[], skipped[]}`, com valor antigo e novo de cada
  propriedade alterada — uma alteração que remove constraint passa a dizê-lo.
- **Correção de fato:** `_conf.superEntity` e `_conf.superEntityStrategy` **são** atualizáveis; o BNF
  os listava como imutáveis.
- **Chave única: `attribute.unique` e `_conf.uniqueKey` são ortogonais e podem coexistir.** Documentado
  o que antes era conhecimento tácito: unicidade de um atributo isolado vs chave composta (2+ atributos).
  Combinar os dois é legítimo quando incidem sobre atributos **diferentes**; quando o **mesmo** atributo
  é `unique` e também está no `uniqueKey`, a composta é logicamente redundante (aceita, mas sem efeito
  prático além do custo). A ordem declarada em `uniqueKey` não é preservada.
- Atualizados: `forger/endpoints/entity.md` (novas seções "Chave única" e "Semântica do `PUT`: ausente
  vs vazio" + corpo da resposta), `forger/bnfs/update-entity-request.bnf`, `forger/openapi.yaml`
  (`EntityPartialUpdate`, `ChangeReport`, `ChangeEntry`).

## 1.12 — 2026-07-05

- **Contrato de identidade `id`: projeção (Long) vs agregado (UUID).** A projeção (read model) tem DOIS
  identificadores — `id` (**Long**, PK da linha) e `aggregateid` (**UUID**, o id do agregado;
  `projecao.aggregateid == aggregate.id`). Os endpoints persistence-crs (`/a/{bc}/{type}/{id}`:
  comando/transição, leitura de estado, `/history`) exigem o **UUID** (`aggregateid`); enviar a PK `id`
  (Long) → `510` "Invalid UUID string" (achado no E2E `biblioteca`). Atualizados:
  `persistence-crs/spec/model-format.md`, `.../erros.md`, `.../endpoints/agregado-leitura.md`,
  `persistence-q/README.md` + `exemplos.md` (exemplos corrigidos p/ `id`:Long + `aggregateid`:UUID),
  `02-conceitos.md`, `05-antipatterns.md`.

## 1.11 — 2026-06-27

- **orgid: novo campo obrigatório `client` na criação de organização.** `client` = **nome da empresa
  cliente** que contratou a plataforma (toda org deve sinalizar). Obrigatório em `POST /up/org` e no
  registro `POST /open/up/account` (`org.client`); **máx. 12 caracteres** (ausente/vazio ou >12 → `400`).
  Atualizados: `up-organizacao.md`, `publico.md`, `orgid/README` (conceito), `openapi.yaml`, `exemplos.md`.
- No `PUT /up/org`, `client` **não** é listado como editável (é definido na criação). *(Motivo interno,
  fora da doc: a alteração de `client` via PUT não persiste — defeito no serviço a corrigir.)*

## 1.10 — 2026-06-27

- **CORREÇÃO de fato (supersede 1.2/1.3): o `status` do dataschema É um gate operacional.** As versões
  1.2/1.3 documentaram `status` como "rótulo administrativo, sem gate" — **errado**. Verdade (confirmada
  pelo dono): há dois estados que **controlam** operação:
  - **`MODELING`** — esquema editável (criar/alterar `entity`); o modelo **não** é interpretado/executado
    (persistence-crs/persistence-q **não operam**).
  - **`RUNNING`** — esquema congelado (o **forger rejeita** alterar `entity`); persistence-crs/persistence-q
    **interpretam e executam** o modelo.
  - Para editar schema de sistema em operação: `RUNNING → MODELING → editar → RUNNING`.
- Arquivos atualizados: `forger/endpoints/dataschema.md` (gate substitui o parágrafo "sem gate"),
  `forger/endpoints/entity.md` (pré-condição `MODELING`), `persistence-crs/README.md` e
  `persistence-q/README.md` (pré-condição de runtime `RUNNING`), `03-fluxo-de-deploy.md` (passo 7
  "ativar"), `04-walkthrough.md` (passo 7b), `05-antipatterns.md` (gate nos dois sentidos).
- Origem: o gate vive nos serviços que interpretam o modelo (que ignoram `MODELING`); a recusa no forger
  é parte do contrato documentado.
- **Limites de nome atualizados:** dataschema **≤16** (era 12), entity/atributo/associação **≤24** (era
  12), superentidade **≤24** (era 64). Propagado a `forger/README`, `dataschema.md`, `entity.md`,
  `05-antipatterns.md`.
- **Rename:** `autenticacao.md → 06-autenticacao.md` (numeração); refs repontadas (`llms.txt`,
  `04-walkthrough`, `auth/README`, `auth/exemplos`, `orgid/README`).

## 1.9 — 2026-06-26

- **orgid · reescrita dos endpoints (por domínio, detalhe por endpoint).** A doc anterior misturava os
  domínios `/up` e `/ua` num mesmo arquivo e listava só a URL. Substituída por **arquivos por domínio**
  (`up-organizacao`, `up-conta`, `up-associacao`, `up-contrato`, `ua-conta`, `ua-papel`, `publico`),
  cada endpoint numa **subseção própria** com: **método HTTP**, **path-variables** (com significado),
  **cabeçalhos** (`Authorization`; sem `X-Tenant-Id`), **papéis** exigidos, **corpo** de POST/PUT
  (tabela de campos) e **resposta/status**. Índice do `orgid/README.md` e `llms.txt` repontados.
- Removidos os 6 arquivos antigos (`organizacao`, `conta`, `papel`, `associacao`, `contrato`,
  `registro-publico`); referências externas (README raiz, auth) repontadas para `publico.md`.
- Sanitização: omitidos campos que revelam nuvem/credenciais e o campo de bypass por segredo no envio
  de e-mail; identificador de pagamento mantido como termo genérico.
- **`/ua`:** a associação **conta-papel** (`/ua/account-role`) saiu de `ua-papel.md` para arquivo próprio
  `ua-associacao.md` (espelha o `up-associacao.md`); `ua-papel.md` fica só com **papel**.

## 1.8 — 2026-06-23

- **Novo serviço documentado: auth (emissão de token / login)** — `POST /up/sign-in`, `POST /ua/sign-in`,
  `GET /up/sign-in/renew`. Guia + endpoints + erros + exemplos + OpenAPI 3.1. auth **emite** o token de
  acesso (com identidade: usuário, papéis, organizações, tenants); os demais serviços **validam** (CP-12).
- Posicionado como **camada de identidade, par do orgid** (auth emite, orgid administra): 00-plataforma
  (8 serviços), 01-arquitetura, coordenação (CP-12), llms.txt, README.
- **autenticacao.md reconciliado:** a emissão do token agora aponta para o **auth** (não mais "fora de escopo").
- Sanitização: lib de token/criptografia, prefixos ofuscados, headers de rastreamento, padrão de sessão,
  hosts e segredos de token não documentados.

## 1.7 — 2026-06-23

- **forger (código): normalização de `schema.forWriteModel.name` na publicação do `.model.json`.** O
  armazém de escrita é universal; o forger impõe o valor canônico — se o JSON enviado não contiver um
  dos dois valores válidos, é convertido automaticamente (e, se ausente, definido). Doc atualizada
  (model-format, forger/model) — sem expor os valores reais (representados como `wdb.client`).

## 1.6 — 2026-06-23

- **Novo serviço documentado: filer (arquivos)** — upload/download/listar/remover de arquivos anexos a
  agregados. Guia + 4 endpoints + erros + exemplos + OpenAPI 3.1. Auth `Authorization`+`X-Tenant-Id`;
  RBAC por entidade (leitura/escrita). Chave `{org}-{project}-{entity}-{entityId}-{attribute}.{ext}`
  amarra o arquivo ao agregado; um agregado pode ter vários arquivos (por atributo).
- Posicionado como **serviço transversal**: 00-plataforma (7 serviços), 01-arquitetura, coordenação
  (CP-11), llms.txt, README.
- Sanitização: armazenamento de objetos/cache/infra abstraídos; `code` (chave de função) e detalhes
  de nuvem/rede não documentados.

## 1.5 — 2026-06-23

- **Novo serviço documentado: orgid (IAM)** — identidade e acesso (organizações, contas, papéis/RBAC,
  contratos). Guia + endpoints (organização, conta, papel, associação conta-papel-org, contrato,
  registro público `/open/*`) + erros + exemplos + OpenAPI 3.1. Dois domínios (`/up` plataforma, `/ua`
  externo) + público por design (`/open`). Valida token (não emite).
- Posicionado como **pré-requisito de identidade** (antes do forger): 00-plataforma (6 serviços),
  01-arquitetura (Fase 0), 03-fluxo-de-deploy (passo 0), coordenação (CP-8/CP-9/CP-10), llms.txt, README.
- Excluídos (não-públicos): endpoints internos de uso entre serviços (prefixos ofuscados), de teste e
  de observabilidade/administração.

## 1.4 — 2026-06-20

- **Convenção `_`-prefixo (metadados):** chaves de topo do `.model.json` começando com `_` (ex.:
  `_comment`, `_meta`, `_schemaVersion`) são metadados não-semânticos — o forger as **remove** (validação
  + JSON) e o grid de interpretação ignora. Use `_comment` para comentário livre; `comment` sem prefixo
  no topo quebra o forger (confundido com a chave do bounded context).
- Exemplos: `comment`/`$comment` de topo → `_comment` nos 6 modelos.
- Schema/`model-format.md`: reconhecem chaves de topo `_`-prefixadas; nota da exceção `attribute.comment`
  (semântico → comentário de coluna; permanece sem `_`).
- forger: o serviço de cache de modelo remove **recursivamente** toda chave `_`-prefixada (qualquer nível)
  (antes só `_schemaVersion`/`_meta`, top-level). Risco avaliado: nenhum consumidor lê `_x` do modelo. Requer rebuild/deploy.

## 1.3 — 2026-06-20

- **`persistence-q/query-controls.md`** (NOVO, verificado no código): operadores de predicado
  (`eq/neq/gt/gte/lt/lte/like/ilike/in`), `_connective` (AND/OR), `_paging`
  (`_maxRegisters`/`_firstRegister`), `_sorting` (`_orderBy`/`_order`), `_count`, `_cache`
  (`_behavior`: use/evict/ignore), e controles de associação (`_associations`/`_populating`/`_level`/`_as`).
  Era o maior gap real do corpus. Indexado em `consulta.md`, `README`, `llms.txt`.
- Itens da proposta (DEMANDA 2026-06-20) **não** portados por não se confirmarem no código: ciclo de
  estado do dataschema (MODELING→RUNNING→STOPPED), prereq `dataschema=MODELING` para entity, e
  "type-mapping enum" no forger (forger usa os 8 tipos direto; Decimal→Long/uuid→String(36) são
  convenções de autoria, não regra do serviço).

## 1.2 — 2026-06-20

- **OpenAPI** (3.1) por serviço: `forger/openapi.yaml`, `persistence-crs/openapi.yaml`,
  `persistence-q/openapi.yaml`, `br-service/openapi.yaml` (tech-agnóstico; só `Authorization`/`X-Tenant-Id`).
- **`05-antipatterns.md`**: o que mais quebra, por serviço (sintoma → causa → correção).
- `model-format.md`: identificação na projeção (linha por `aggregateid`; `status` = estado atual).
- Fronteira de publicação: a allowlist passa a incluir `.yaml`.
- `dataschema.md`: `status` documentado como **rótulo administrativo editável** (sem lifecycle imposto
  nem gate para criar entity). O ciclo "MODELING→RUNNING→STOPPED" da proposta **não** existe no código
  (drift do mirror); `ACTIVE` é status de broker, não de dataschema.

## 1.1 — 2026-06-20

- Guardião dividido: `docs/CLAUDE.md` (sensível, não publicado) + `public/CLAUDE-extended.md` (público).
- `llms.txt` como índice primário; `llms-full.txt` como fallback (cabeçalho + ordem alto-valor-primeiro).
- Regras contexto-aware (teto de tamanho, fatos-chave no topo, ordenação) no guardião.
- **Fronteira de publicação:** definida a allowlist do que é publicado e a denylist do que nunca sai,
  com as guardas de verificação antes de disponibilizar.

## 1.0 — 2026-06-20

Primeira versão consolidada da documentação pública para agentes de IA.

- Documentados os cinco serviços (forger, persistence-crs, es-n, persistence-q, br-service): guia,
  endpoints, erros e exemplos, com ciclo de vida de cada requisição.
- Documentos transversais: conceitos, composição, arquitetura ponta a ponta, fluxo de deploy,
  walkthrough único, autenticação/tenant, coordenação (CP-1…CP-7).
- **Gramática formal do `.model.json`** ([model-format.md](persistence-crs/spec/model-format.md)) +
  **JSON Schema** ([model.schema.json](persistence-crs/spec/model.schema.json)); os 6 modelos de
  exemplo validam contra o schema.
- Gramáticas **BNF** dos payloads do forger ([forger/bnfs/](forger/bnfs/README.md)).
- Índices: [`llms.txt`](llms.txt) (navegável) e [`llms-full.txt`](llms-full.txt) (corpus num arquivo).
- Sanitização **tech-agnóstica total**: sem nomes de produto/stack/versão, classes, filas, hosts,
  cabeçalhos internos, endpoints sem autenticação, UUIDs reais ou nomes de cliente.

> Para regenerar `llms-full.txt` após editar qualquer `.md`, concatene os arquivos na ordem do
> `llms.txt`.
