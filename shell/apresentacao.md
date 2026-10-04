# shell · como preencher e publicar o manifesto de apresentação

> Para **quem escreve** o manifesto de apresentação de um tenant — a hierarquia de agentes que já escreve
> o `.model.json`. Diz o que cada chave **faz na tela** do Stager, como escolher o valor, e como publicar
> e conferir. A **forma** do arquivo e o que a publicação recusa estão em
> [forger — manifesto de apresentação](../forger/endpoints/presentation.md); o que a casca entrega ao
> miolo, em [contrato-miolo](contrato-miolo.md). Pré: [README](README.md).

## Receita: do id ao nome

Para a tela mostrar **nome** onde o registro guarda o **id** de outro agregado, e pedir **busca** em vez de
id colado, são três declarações e duas publicações:

1. **No `.model.json` de quem aponta**, diga para onde o atributo aponta, com `references` no nível do
   agregado:
   `"references": { "teacherid": "cadastro.professor" }` quando o alvo está no **mesmo modelo**, ou
   `"references": { "teacherid": "cadastro.cadastro.professor" }` (`<projeto>.<bc>.<tipo>`) quando está em
   **outro projeto do mesmo org**. Forma e recusas:
   [forger — `references`](../forger/endpoints/model.md#atributo-que-aponta-outro-agregado-references).
2. **No manifesto do agregado APONTADO**, diga qual atributo é o nome dele: `"titleKey": "fullname"`. Sem
   isso a tela usa o `identity.fields` do modelo dele e, por fim, o primeiro texto que ele declara.
3. **No manifesto de quem aponta**, dê o rótulo do campo: `"labels": { "teacherid": "Professor" }`. Sem
   isso o campo aparece como "Teacherid".
4. **Publique o modelo, depois o manifesto** ([Como publicar e conferir](#como-publicar-e-conferir)) e abra
   o agregado de novo no menu.

O que muda na tela está em [Referência a outro agregado](#referência-a-outro-agregado). Se aparecer "sem
acesso", o usuário não tem o tenant do alvo no token ou não tem papel de leitura nele; "não encontrado", o
id não está na projeção do alvo.

## Por que escrever um

Sem manifesto a tela funciona, mas fala a língua do modelo: o rótulo é o nome técnico humanizado
(`emailcorporativo` vira "Emailcorporativo"), a data sai como data e o número como número — sem moeda,
sem telefone, sem saber qual campo identifica o registro. O manifesto é o que faz a tela falar a língua
de quem a usa. **Não é domínio**: o motor não o lê, e nada nele muda o que um comando aceita.

- **Um manifesto por tenant**, no mesmo escopo do `.model.json` daquele tenant.
- **Escreva-o junto com o modelo**, e **republique-o quando o modelo mudar**: referência a atributo que
  deixou de existir a tela ignora, e a próxima publicação recusa.
- Tudo é **opcional**. O que faltar, a tela deriva do nome e do tipo.

## O que cada chave faz na tela

A chave de `aggregate` é a **mesma** do `.model.json` (`<boundedContext>.<tipo>`).

| Chave | Onde aparece | Como escolher |
|---|---|---|
| `singular` | botão **"Novo {singular}"**, estado vazio ("Nenhum registro de {singular} ainda"), nome do arquivo exportado (`{singular}-AAAA-MM-DD.csv`) | em **minúsculas**, porque entra no meio da frase: `"aluno"`, `"pedido"` |
| `plural` | **título da página** do agregado; elemento raiz do XML exportado | com inicial maiúscula: `"Alunos"` |
| `genero` | a concordância do botão de criação e do estado vazio: **"Nova** modalidade", "criar **a primeira**" | `"feminino"` quando o nome do agregado é feminino; sem a chave, a tela diz "Novo" |
| `titleKey` | título do registro no **painel lateral**, no **cartão** do celular e em "Ações deste registro"; e o **nome** do registro onde outro agregado o aponta (ver [Referência a outro agregado](#referência-a-outro-agregado)) | o atributo (ou `grupo.campo`) que uma pessoa usa para reconhecer o registro: nome, número, placa. Sem ele, a tela usa o primeiro texto do registro |
| `labels` | cabeçalho da coluna, rótulo do filtro, rótulo do campo no formulário, linhas do histórico, aba Dados, cópia de linha | para **todo** atributo que aparece na tela. Value object pelo nome = título do grupo no formulário; `grupo.campo` = campo dentro dele |
| `stateLabels` | a **pílula de estado** (tabela, painel, ações, formulário, histórico) | o estado como o usuário diz: `"matriculado"` → `"Matriculado"` |
| `fmt` | como o valor é **mostrado** na tabela, no cartão, na aba Dados e no histórico — também de `grupo.campo` | ver [formatos](#formatos). Só muda a exibição: o comando e a exportação continuam com o valor cru |
| `cols` | as colunas visíveis **por padrão**, na ordem — `grupo.campo` vira coluna | de 4 a 6, as que se lê para decidir. **Não** ponha `status` (o Estado é sempre a última coluna) nem campo técnico. O usuário pode mudar as colunas; "Restaurar padrão" volta a esta lista |
| `filters` | os campos do **painel de filtros** — `grupo.campo` filtra dentro do grupo | os que se usa para achar registro. Sem a chave, todos os atributos aparecem no filtro (campos de grupo, não) |
| `options` | lista fixa: o campo do formulário vira **seleção**. Referência: o campo mostra o **nome** do registro apontado em toda a tela e se preenche por **busca** — atributo ou campo de grupo | lista fixa: os **valores exatos** que o comando grava. Referência `{ aggregate, valueKey, labelKey }`: `valueKey` é o atributo do outro agregado que **este** campo guarda (num campo de referência, costuma ser `aggregateid`); `labelKey`, o que a tela mostra. Ver [Referência a outro agregado](#referência-a-outro-agregado) |
| `stateHue` | a **cor** da pílula de cada estado (matiz de 0 a 359) | para dar à cor um significado: em torno de 150 verde, 25 vermelho, 250 azul, 80 amarelo. Sem ela, a cor sai do nome do estado e é a mesma em qualquer tela |

### Campos de value object: `grupo.campo`

Um value object guarda um grupo de campos num atributo só: o **endereço** (cidade, CEP) de um cadastro, as
**matrículas** (plano, preço) de um aluno. `grupo.campo` aponta um campo de dentro do grupo, e a tela o usa
em `labels`, `titleKey`, `fmt`, `cols`, `filters` e `options`:

- grupo **único** (`single`): o campo do objeto — a coluna "Cidade" mostra `Natal`;
- grupo **com vários itens** (`multiple`): os valores de todos os itens, separados por vírgula — a coluna
  "Plano" mostra `Mensal, Anual`; no filtro, o registro entra se **algum** item casar;
- na aba Dados, cada campo do grupo aparece numa linha própria, com o rótulo e o formato de `grupo.campo`.

Sem manifesto, campo de grupo não entra nas colunas padrão nem no filtro: só aparece se o manifesto pedir.

**Não entra no manifesto** o nome de exibição de **comando** e de **evento**: é o `alias`, declarado no
próprio `.model.json` ([forger — model](../forger/endpoints/model.md#nome-de-exibicao)).

### Formatos

| `fmt` | Mostra | Para |
|---|---|---|
| `date` | `26/09/2026` | `Date`, ou texto `AAAA-MM-DD` |
| `datetime` | `26/09/2026 14:30`, no fuso de quem lê | `Timestamp` |
| `money` | `R$ 199,00` a partir de `199` | valor em reais |
| `money:<casas>` | `R$ 199,00` a partir de `19900` com `money:2` | inteiro com casas implícitas — o caso comum de centavos em `Long` |
| `phone` | `(84) 99999-0001` | 10 ou 11 dígitos; outro tamanho sai como veio |
| `bool` | `Sim` / `Não` | `Boolean` |
| `mono` | o valor em fonte monoespaçada | documentos, códigos, identificadores |

Sem `fmt`, `Date` e `Timestamp` já saem formatados, `Boolean` sai como Sim/Não e número sai em
monoespaçada. `fmt` serve sobretudo para **dinheiro** e **telefone**, que o tipo não revela.

### Referência a outro agregado

O campo **aponta outro agregado** quando o modelo o declara em `references` (a forma recomendada, ver a
[receita](#receita-do-id-ao-nome)) ou quando o manifesto traz `options` na forma
`{ aggregate, valueKey, labelKey }`. A aula aponta o professor; o aluno, nas matrículas, aponta o plano. A
tela deriva o resto:

- **onde se vê** — tabela, cartão, aba Dados, histórico (antes → depois), título, cópia e filtro mostram o
  **nome** do registro apontado, e não o id. A exportação leva as duas coisas: a coluna do id e, ao lado,
  `<atributo>_rotulo`;
- **onde se preenche** — no formulário (atributo ou campo de grupo) e no filtro ("é"), o campo vira um
  **seletor com busca**, página a página, sem teto de registros. Grava `valueKey`, nunca o nome;
- **qual nome** — `labelKey`; se ele faltar, ou se o papel não o ler, o `titleKey` do agregado apontado;
  depois, o `identity.fields` dele no modelo; por fim, o primeiro texto que ele declara. A busca procura
  nesses atributos de texto;
- **quando não há nome** — a tela diz o motivo: "não encontrado" (o id não está na projeção), "sem
  acesso" (o papel não lê o agregado apontado) ou "rótulo indisponível" (falha, registrada no diagnóstico).
  **O seletor não aceita texto livre**: se a lista não vem, o erro aparece no campo.

**Duas formas de declarar, e qual usar.**

| Forma | Onde | Alcança | Quando usar |
|---|---|---|---|
| `references` | `.model.json`, nível do agregado | o próprio modelo (`<bc>.<tipo>`) e outro projeto do org (`<projeto>.<bc>.<tipo>`) | **sempre que puder**: é fato do domínio e vale também para cliente sem tela |
| `options` por referência | manifesto | só o mesmo modelo | quando não se pode republicar o modelo, ou para fixar um `labelKey` diferente do `titleKey` do alvo |

Para o mesmo atributo, `references` vale sobre o `options`. Quem resolve é o BFF
([bff — referência entre agregados](../bff/README.md#referência-entre-agregados-sessionrefs)), com o token de
quem usa a tela. A referência vive em **atributo ou `grupo.campo`**, nunca dentro de um valor `Json`, onde a
tela não tem o que interpretar.

## Exemplo

```json
{
  "formato": 1,
  "aggregate": {
    "cadastro.aluno": {
      "singular": "aluno",
      "plural": "Alunos",
      "titleKey": "fullname",
      "labels": { "fullname": "Nome completo", "cpf": "CPF", "phone": "Telefone", "birthdate": "Nascimento", "plano": "Plano" },
      "stateLabels": { "criado": "Criado", "matriculado": "Matriculado", "cancelado": "Cancelado" },
      "fmt": { "cpf": "mono", "phone": "phone", "birthdate": "date" },
      "cols": ["fullname", "cpf", "phone", "birthdate"],
      "filters": ["fullname", "cpf", "status"],
      "options": { "gender": ["Feminino", "Masculino", "Outro"], "plano": { "aggregate": "cadastro.plano", "valueKey": "codigo", "labelKey": "name" } },
      "stateHue": { "matriculado": 155, "cancelado": 25 }
    }
  }
}
```

> Ilustrativo: nomes de agregado, atributo e estado são exemplos, não JSON literal a copiar.

## Como publicar e conferir

1. **Publicar** com a **conta de plataforma**, a mesma autorização do `.model.json`:
   `POST /org/{org}/project/{project}/tenant/{tenantId}/presentation` no forger. A publicação confere a
   forma e cada referência contra o modelo **já publicado** do tenant — publique o modelo primeiro.
   Detalhe e erros: [forger — manifesto de apresentação](../forger/endpoints/presentation.md).
2. **Conferir o que ficou gravado**: `GET` no mesmo caminho do forger. Lê quem tem o tenant no token
   **ou** quem tem papel de administrador ou de engenheiro na org — é assim que a conta que publica confere
   o que publicou ([forger — ler](../forger/endpoints/presentation.md)); `204` = não há manifesto. Pelo
   BFF, com um usuário do app: `GET /session/presentation?tenant={tenantId}`
   ([BFF](../bff/README.md#manifesto-de-apresentação)). O `201` da publicação já diz que a forma e as
   referências passaram.
3. **Ver na tela**: a casca lê o manifesto quando o agregado é aberto. Depois de publicar, abra o
   agregado de novo no menu.

## Checklist de quem escreve

- [ ] A chave do agregado é a do `.model.json`, e todo atributo citado existe no modelo publicado.
- [ ] `singular` em minúsculas; `plural` com inicial maiúscula; `genero: "feminino"` quando o nome é feminino.
- [ ] `titleKey` é um atributo que identifica o registro para uma pessoa — é também o nome dele onde outro agregado o aponta.
- [ ] Todo campo que guarda o id de outro agregado está em `references` no modelo (ou tem `options` por referência): sem isso, a tela mostra e pede o id.
- [ ] Todo agregado que é apontado por outro tem `titleKey`.
- [ ] Todo atributo que aparece na tela tem `labels`; todo estado tem `stateLabels`.
- [ ] Dinheiro em centavos usa `money:2`; telefone usa `phone`.
- [ ] `cols` tem de 4 a 6 atributos, sem `status` e sem campo técnico.
- [ ] Textos em português, diretos, sem jargão para o usuário final.
- [ ] O `alias` de comando e de evento ficou no `.model.json`, não aqui.
