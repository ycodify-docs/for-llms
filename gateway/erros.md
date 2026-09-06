# gateway — catálogo de erros da borda

> Toda resposta abaixo é **produzida pela borda**: o serviço de destino **não foi chamado**. Se o
> erro que você recebeu está aqui, procurar o defeito no serviço é procurar no lugar errado.
> Voltar ao [guia da borda](README.md).

## Como saber que a resposta veio da borda

Respostas geradas aqui trazem cabeçalhos que o serviço não emite. **Não é um cabeçalho só — cada
verificação assina de um jeito**, e procurar o cabeçalho errado leva à conclusão errada:

| Cabeçalho | Aparece em | Significa |
|---|---|---|
| `X-Blocked-Reason` | rota não reconhecida, origem não autorizada, endpoint administrativo negado | por que foi encerrada |
| `X-Blocked-By` | rota não reconhecida, validação de requisição | qual verificação encerrou |
| `X-Security-Warning` | validação de requisição | a validação recusou |
| `X-Size-Limit-Exceeded` · `X-Max-Size-Allowed` · `X-Actual-Size` | **corpo ou cabeçalhos grandes demais** | o teto e o tamanho recebido |
| `X-RateLimit-…` | **vazão excedida e origem banida** | o limite atingido e o motivo |

> ⚠️ **Não procure `X-Blocked-By` numa resposta de tamanho ou de vazão — ele não está lá.** Um `413`
> da borda se reconhece por `X-Size-Limit-Exceeded`; um `429` ou um `403` de banimento, por
> `X-RateLimit-…`. Concluir "não tem `X-Blocked-By`, logo não foi a borda" é o erro mais fácil de
> cometer com este catálogo.

**A regra geral que vale sempre:** se a resposta traz **qualquer** cabeçalho `X-` da tabela acima,
foi a borda. Nenhum serviço os emite.

> **Corpo da resposta.** Desde 2026-09-04, quando a recusa é **erro de quem chamou** — cabeçalho
> ausente, cabeçalhos conflitantes, formato inválido —, **o motivo vem no corpo e também em
> `X-Blocked-Reason`**. Quando a recusa é **detecção de ataque**, o corpo continua opaco e o motivo
> **não** é revelado: dizer ao atacante qual padrão disparou a defesa é ajudá-lo a contorná-la.
> Documentação que descreva as recusas da borda como "corpo sempre vazio" está desatualizada.

---

## 401 — falta o cabeçalho de identificação

| Situação | Correção |
|---|---|
| não veio nem `X-Tenant-Id` nem `X-Forger-Credential` | mande **um** dos dois |
| rota de execução sem `X-Tenant-Id` | rotas de persistência, coordenação e arquivos exigem `X-Tenant-Id` |
| rota administrativa sem `X-Forger-Credential` | as demais rotas exigem `X-Forger-Credential` |

Ver a tabela de qual rota exige qual em [README.md](README.md#cabeçalhos-de-identificação--exatamente-um-nunca-dois).

## 400 — cabeçalhos conflitantes, formato inválido, ou conteúdo recusado

| Situação | Como reconhecer | Correção |
|---|---|---|
| **`X-Tenant-Id` e `X-Forger-Credential` enviados juntos** | motivo no corpo e em `X-Blocked-Reason` | mande **exatamente um**. É o erro mais comum contra a borda |
| formato do tenant ou da credencial fora do padrão | motivo no corpo | conferir o valor; a borda valida a forma antes de encaminhar |
| conteúdo com padrão de injeção (SQL, script, travessia de caminho, caracteres de controle) | **só `X-Security-Warning`**, sem motivo | revisar o que está sendo enviado. **O corpo não dirá qual padrão disparou** |

## 403 — três origens diferentes, e os cabeçalhos as separam

| Origem | Como reconhecer |
|---|---|
| **origem não autorizada** a acessar a plataforma | `X-Blocked-Reason` |
| **endpoint administrativo** chamado de origem não autorizada | `X-Blocked-Reason` |
| **origem banida** por reincidência (ver abaixo) | `X-RateLimit-…` — **não** `X-Blocked-*` |
| caminho marcado como não protegido | `X-Security-Warning` |

> **`403` sem nenhum desses cabeçalhos não veio da borda** — veio do servidor de entrada à frente
> dela, que recusa caminhos sem rota publicada **antes** de a requisição chegar aqui. É um sintoma
> diferente, com correção diferente: falta publicar a rota lá, não aqui.

## 404 — a borda não reconheceu a rota

**Este é o `404` que mais engana**, porque é indistinguível, à primeira vista, do `404` de um recurso
inexistente dentro do serviço. O corpo é genérico e **não revela** quais rotas existem.

| Causa | Correção |
|---|---|
| segundo segmento do caminho não é um serviço conhecido | conferir o nome do serviço no caminho |
| **falta um segmento obrigatório** depois do serviço | vários serviços exigem um terceiro segmento; sem ele, a borda recusa **antes** de chegar ao serviço |
| a rota não existe **naquele ambiente** | a mesma chamada pode funcionar em outro ambiente |

> **Regra de bolso:** `404` **com** `X-Blocked-By` e `X-Blocked-Reason` veio da borda e o serviço
> nunca soube da requisição. `404` **sem** esses cabeçalhos veio do serviço, e aí sim o recurso é que
> não existe.

## 413 — corpo ou cabeçalhos grandes demais

Reconhece-se por **`X-Size-Limit-Exceeded: true`**, acompanhado de `X-Max-Size-Allowed` e
`X-Actual-Size` — que dizem, em números, o teto e o que foi recebido.

O teto **varia por serviço de destino**, e não é um número único da plataforma. Serviços que recebem
carga binária (publicação de artefatos, anexos) têm teto próprio, maior.

> **Caminho que a borda não reconhece cai no teto mais restritivo.** Um `413` inesperado em rota nova
> costuma ser sintoma de **rota não publicada**, não de payload realmente grande.

Um `413` **sem** `X-Size-Limit-Exceeded` veio do servidor de entrada à frente da borda, não daqui.

## 431 — cabeçalhos grandes demais

A soma dos cabeçalhos excedeu o teto. Costuma ser efeito de token muito grande ou de acúmulo de
cabeçalhos de rastreamento adicionados em cadeia.

## 429 — vazão excedida

Reconhece-se pelos cabeçalhos `X-RateLimit-…`. Dois limites independentes, mesma resposta:

| Limite | Quando aparece |
|---|---|
| por **origem** | proteção de infraestrutura contra volume anômalo |
| por **tenant** | limite da aplicação, configurável |

Reduza a taxa. Se o volume é legítimo e recorrente, o limite por tenant é ajustável — o por origem
não deve ser contornado.

> **Insistir depois do `429` muda o código, não só o corpo.** Origem que continua chamando passa a
> receber **`403`** (banimento), também com `X-RateLimit-…`. Quem só trata `429` no cliente vê o
> erro "mudar de natureza" sem explicação.

## Reincidência — banimento por tempo

Origens que insistem em **caminhos inexistentes** são marcadas como suspeitas e, na sequência,
**banidas por 7 dias** (padrão configurável). Um cliente legítimo não chega lá; um varredor, sim.

Se um ambiente de testes começar a receber recusa em tudo depois de uma bateria de chamadas a
caminhos errados, é este mecanismo. A correção é **parar de gerar os caminhos errados** — não
aumentar limite.

## 503 — serviço indisponível

O serviço de destino não está registrado, ou o **disjuntor** daquele serviço está aberto após falhas
sucessivas. A borda respondeu porque não havia a quem encaminhar. **Vem com corpo em JSON** dizendo
qual serviço e qual caminho — diferente das recusas de segurança, que são deliberadamente econômicas.

---

## Erro de origem cruzada no navegador

Sintoma: o navegador recusa a resposta alegando **mais de um valor** para o cabeçalho de origem
permitida.

Causa: o serviço emitiu os seus próprios cabeçalhos de origem cruzada **além** dos da borda. A borda
**remove** os do serviço e aplica os seus antes de a resposta ser escrita — mas serviço que os emita
em caminho não coberto ainda produz duplicidade.

Correção: **o serviço para de emitir cabeçalhos de origem cruzada.** Não é ajuste na borda.

---

## Serviços que **não** passam pela borda

Nem todo serviço da plataforma fica atrás da borda. O de **arquivos estáticos** responde por um
domínio próprio, direto no servidor de entrada.

**Consequência para o diagnóstico:** uma resposta dele **nunca** traz cabeçalho da borda — e a
ausência deles ali **não** significa "a borda deixou passar", significa que **a borda nunca viu a
requisição**. Não generalize a regra de bolso deste catálogo para domínios que não sejam o da API.
