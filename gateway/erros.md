# gateway — catálogo de erros da borda

> Toda resposta abaixo é **produzida pela borda**: o serviço de destino **não foi chamado**. Se o
> erro que você recebeu está aqui, procurar o defeito no serviço é procurar no lugar errado.
> Voltar ao [guia da borda](README.md).

## Como saber que a resposta veio da borda

Respostas geradas aqui trazem cabeçalhos que o serviço não emite:

| Cabeçalho | Significa |
|---|---|
| `X-Blocked-By` | a verificação que encerrou a requisição |
| `X-Blocked-Reason` | por que foi encerrada |
| `X-Security-Warning` | a validação de segurança recusou a requisição |

**Na dúvida, olhe os cabeçalhos de resposta antes de abrir chamado com o time do serviço.**

> **Corpo da resposta.** Desde 2026-09-04, quando a recusa é **erro de quem chamou** — cabeçalho
> ausente, cabeçalhos conflitantes, formato inválido —, **o motivo vem no corpo**. Quando a recusa é
> **detecção de ataque**, o corpo continua opaco de propósito: dizer ao atacante qual padrão disparou
> a defesa é ajudá-lo a contorná-la. Documentação que descreva as recusas da borda como "corpo
> sempre vazio" está desatualizada.

---

## 401 — falta o cabeçalho de identificação

| Situação | Correção |
|---|---|
| não veio nem `X-Tenant-Id` nem `X-Forger-Credential` | mande **um** dos dois |
| rota de execução sem `X-Tenant-Id` | rotas de persistência, coordenação e arquivos exigem `X-Tenant-Id` |
| rota administrativa sem `X-Forger-Credential` | as demais rotas exigem `X-Forger-Credential` |

Ver a tabela de qual rota exige qual em [README.md](README.md#cabeçalhos-de-identificação--exatamente-um-nunca-dois).

## 400 — cabeçalhos conflitantes, formato inválido, ou conteúdo recusado

| Situação | Correção |
|---|---|
| **`X-Tenant-Id` e `X-Forger-Credential` enviados juntos** | mande **exatamente um**. Este é o erro mais comum contra a borda |
| formato do tenant ou da credencial fora do padrão esperado | conferir o valor; a borda valida a forma antes de encaminhar |
| conteúdo com padrão de injeção (SQL, script, travessia de caminho, caracteres de controle) | revisar o que está sendo enviado. **O corpo não dirá qual padrão disparou** |

## 403 — caminho proibido

Caminhos marcados como não protegidos são recusados por definição, em qualquer ambiente.

## 404 — a borda não reconheceu a rota

**Este é o `404` que mais engana**, porque é indistinguível, à primeira vista, do `404` de um recurso
inexistente dentro do serviço. O corpo é genérico e **não revela** quais rotas existem.

| Causa | Correção |
|---|---|
| segundo segmento do caminho não é um serviço conhecido | conferir o nome do serviço no caminho |
| **falta um segmento obrigatório** depois do serviço | vários serviços exigem um terceiro segmento; sem ele, a borda recusa **antes** de chegar ao serviço |
| a rota não existe **naquele ambiente** | a mesma chamada pode funcionar em outro ambiente |

> **Regra de bolso:** `404` **com** `X-Blocked-By` veio da borda e o serviço nunca soube da
> requisição. `404` **sem** esses cabeçalhos veio do serviço, e aí sim o recurso é que não existe.

## 413 — corpo grande demais

O teto **varia por serviço de destino**, e não é um número único da plataforma. Serviços que recebem
carga binária (publicação de artefatos, anexos) têm teto próprio, maior.

> **Cuidado ao diagnosticar:** um `413` pode vir da borda **ou** do proxy à frente dela, e os dois
> são indistinguíveis pelo código. Os cabeçalhos de resposta separam: os da borda trazem
> `X-Blocked-By`.

Caminho que a borda não reconhece cai no **teto mais restritivo**. Ou seja, um `413` inesperado em
rota nova costuma ser sintoma de rota não publicada, não de payload realmente grande.

## 429 — vazão excedida

Dois limites independentes, e a resposta é a mesma:

| Limite | Quando aparece |
|---|---|
| por **origem** | proteção de infraestrutura contra volume anômalo |
| por **tenant** | limite da aplicação, configurável |

Reduza a taxa. Se o volume é legítimo e recorrente, o limite por tenant é ajustável — o por origem
não deve ser contornado.

## 431 — cabeçalhos grandes demais

A soma dos cabeçalhos excedeu o teto. Costuma ser efeito de token muito grande ou de acúmulo de
cabeçalhos de rastreamento adicionados em cadeia por proxies.

## 503 — serviço indisponível

O serviço de destino não está registrado no **registro de serviços**, ou o **disjuntor** daquele
serviço está aberto após falhas sucessivas. A borda respondeu porque não havia a quem encaminhar.

---

## Erro de origem cruzada no navegador

Sintoma: o navegador recusa a resposta alegando **mais de um valor** para o cabeçalho de origem
permitida.

Causa: o serviço emitiu os seus próprios cabeçalhos CORS **além** dos da borda. A borda gerencia CORS
de forma centralizada e remove os do serviço — mas serviço que os emita em caminho não coberto ainda
produz duplicidade.

Correção: **o serviço para de emitir CORS.** Não é ajuste na borda.
