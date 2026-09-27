# shell · como preencher e publicar o manifesto de apresentação

> Para **quem escreve** o manifesto de apresentação de um tenant — a hierarquia de agentes que já escreve
> o `.model.json`. Diz o que cada chave **faz na tela** do Stager, como escolher o valor, e como publicar
> e conferir. A **forma** do arquivo e o que a publicação recusa estão em
> [forger — manifesto de apresentação](../forger/endpoints/presentation.md); o que a casca entrega ao
> miolo, em [contrato-miolo](contrato-miolo.md). Pré: [README](README.md).

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
| `titleKey` | título do registro no **painel lateral**, no **cartão** do celular e em "Ações deste registro" | o atributo que uma pessoa usa para reconhecer o registro: nome, número, placa. Sem ele, a tela usa o primeiro texto do registro |
| `labels` | cabeçalho da coluna, rótulo do filtro, rótulo do campo no formulário, linhas do histórico, aba Dados, cópia de linha | para **todo** atributo que aparece na tela. Value object pelo nome = título do grupo no formulário; `grupo.campo` = campo dentro dele |
| `stateLabels` | a **pílula de estado** (tabela, painel, ações, formulário, histórico) | o estado como o usuário diz: `"matriculado"` → `"Matriculado"` |
| `fmt` | como o valor é **mostrado** na tabela, no cartão, na aba Dados e no histórico | ver [formatos](#formatos). Só muda a exibição: o comando e a exportação continuam com o valor cru |
| `cols` | as colunas visíveis **por padrão**, na ordem | de 4 a 6, as que se lê para decidir. **Não** ponha `status` (o Estado é sempre a última coluna) nem campo técnico. O usuário pode mudar as colunas; "Restaurar padrão" volta a esta lista |
| `filters` | os campos do **painel de filtros** | os que se usa para achar registro. Sem a chave, todos os atributos aparecem no filtro |
| `options` | o campo do formulário vira **seleção** | lista fixa: os **valores exatos** que o comando grava. Referência `{ aggregate, valueKey, labelKey }`: a tela consulta até 200 registros do outro agregado, mostra `labelKey` e grava `valueKey` — então `valueKey` é o atributo do outro agregado que **este** campo guarda: num campo de referência, costuma ser `aggregateid` (o que a produção aceita, ver a página do forger) |
| `stateHue` | a **cor** da pílula de cada estado (matiz de 0 a 359) | para dar à cor um significado: em torno de 150 verde, 25 vermelho, 250 azul, 80 amarelo. Sem ela, a cor sai do nome do estado e é a mesma em qualquer tela |

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
   o que publicou ([forger — ler](../forger/endpoints/presentation.md)). Pelo BFF, com um usuário do app:
   `GET /session/presentation?tenant={tenantId}` ([BFF](../bff/README.md#manifesto-de-apresentação)).
   ⚠️ **Até o próximo deploy do composer**, a conta de plataforma ainda recebe `403` `tenant <id> is not in
   the token` no `GET` — medido no stager em 2026-09-27T00:09Z com o token de plataforma do conceptnatal,
   que lia o `/model` do mesmo tenant com `200`. Até lá, confira com um usuário do app ou pela tela; o `201`
   da publicação já diz que a forma e as referências passaram.
3. **Ver na tela**: a casca lê o manifesto quando o agregado é aberto. Depois de publicar, abra o
   agregado de novo no menu.

## Checklist de quem escreve

- [ ] A chave do agregado é a do `.model.json`, e todo atributo citado existe no modelo publicado.
- [ ] `singular` em minúsculas; `plural` com inicial maiúscula.
- [ ] `titleKey` é um atributo que identifica o registro para uma pessoa.
- [ ] Todo atributo que aparece na tela tem `labels`; todo estado tem `stateLabels`.
- [ ] Dinheiro em centavos usa `money:2`; telefone usa `phone`.
- [ ] `cols` tem de 4 a 6 atributos, sem `status` e sem campo técnico.
- [ ] Textos em português, diretos, sem jargão para o usuário final.
- [ ] O `alias` de comando e de evento ficou no `.model.json`, não aqui.
