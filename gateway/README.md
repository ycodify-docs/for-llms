# gateway — guia da borda

> **Papel:** **borda da plataforma** — o único ponto por onde requisições externas entram. Não é um
> serviço de negócio e **não tem endpoint próprio exposto ao cliente**: encaminha para os serviços e,
> antes disso, decide se a requisição sequer chega a eles. Toda resposta que você recebe passou por
> aqui, e **parte delas nasce aqui** — este guia existe para você saber distinguir uma da outra.
> Pré-requisitos: [conceitos](../02-conceitos.md), [autenticação](../06-autenticacao.md).

## Por que este documento importa

Um erro devolvido pela borda e um erro devolvido pelo serviço chegam ao cliente pelo mesmo canal e,
em vários casos, com o **mesmo código HTTP**. Sem saber separá-los, procura-se o defeito no serviço
errado. O catálogo em [erros.md](erros.md) é a peça central: ele diz, para cada resposta, **quem a
produziu** e o que fazer.

## Caminho de uma requisição

A borda aplica, em ordem, um conjunto de verificações. Cada uma pode encerrar a requisição — e nesse
caso **o serviço de destino nunca é chamado**:

1. **Origem** — endereços não autorizados são recusados antes de qualquer outra coisa.
2. **Rota conhecida** — o caminho precisa casar uma rota publicada. Caminho desconhecido termina em
   `404` produzido pela borda (ver [erros.md](erros.md#404)).
3. **Cabeçalhos de identificação** — regra detalhada abaixo.
4. **Conteúdo malicioso** — tentativas de injeção são recusadas com resposta deliberadamente opaca.
5. **Tamanho** — corpo e cabeçalhos têm teto, e o teto **varia por serviço de destino**.
6. **Vazão** — há limite por endereço de origem e limite por tenant.
7. **Encaminhamento** — só aqui a requisição chega ao serviço, via **registro de serviços**.

Na volta, a borda **normaliza os cabeçalhos de origem cruzada (CORS)**: remove os que o serviço tenha
emitido e aplica os seus. Serviços **não devem** emitir CORS — dois valores para o mesmo cabeçalho
fazem o navegador recusar a resposta inteira.

## Cabeçalhos de identificação — exatamente um, nunca dois

Toda requisição de cliente carrega **um** destes dois cabeçalhos, **nunca os dois juntos e nunca
nenhum**:

| Cabeçalho | O que identifica |
|---|---|
| `X-Tenant-Id` | o tenant cuja sala a requisição acessa |
| `X-Forger-Credential` | a credencial de quem administra recursos |

Qual dos dois depende da rota:

| Rota | Exige |
|---|---|
| rotas de autenticação (`/auth/`) | **um dos dois**, à sua escolha |
| rotas de execução — persistência, coordenação, arquivos | **`X-Tenant-Id`** |
| demais rotas | **`X-Forger-Credential`** |

> **O erro mais comum é mandar os dois.** A intuição de "mando os dois e deixo o servidor escolher"
> falha aqui: a borda recusa com `400`. Ver [erros.md](erros.md#400).

**Exceção — abertura de canal persistente.** O aperto de mão que abre um canal bidirecional de longa
duração não carrega esses cabeçalhos: a autenticação viaja **no primeiro quadro do próprio canal** e
é validada pelo serviço que o atende, não pela borda. A borda deixa passar o aperto de mão; quem
autoriza é o serviço.

## Forma dos caminhos

```
/v{versão}/{serviço}/{resto}
```

A borda conhece uma **lista fechada** de segmentos de serviço. Caminho cujo segundo segmento não
esteja nessa lista recebe `404` **da borda**, com corpo genérico que **não revela** quais serviços
existem — é deliberado, para não entregar o mapa da plataforma a quem sonda.

Alguns serviços exigem um **terceiro segmento** específico; sem ele, o `404` também vem da borda,
ainda que o serviço exista. É a causa mais frequente de "a rota existe, mas dá 404": ver
[erros.md](erros.md#404).

## Reincidência

Origens que insistem em caminhos inexistentes passam a ser **bloqueadas por um período**. Um cliente
legítimo não chega lá; um varredor, sim. Se um ambiente de testes começar a receber recusa em tudo
depois de uma bateria de chamadas a caminhos errados, é este mecanismo — e a correção é parar de
gerar os caminhos errados, não aumentar limite.

## O que **não** está aqui

- **Endpoints administrativos da borda** (configuração de limites, listas de origem, estado dos
  disjuntores) existem, mas são **restritos por origem** e não fazem parte da superfície do cliente.
  Por isso não são documentados aqui: a convenção do repositório cobre apenas endpoints autenticados
  e expostos ao cliente.
- **Arquitetura interna da borda** — organização do código, ordem interna das verificações,
  tecnologia empregada. Fora do escopo desta documentação, que é independente de tecnologia.

## Relacionados

- [Autenticação e tenant](../06-autenticacao.md) — como obter o tenant e o token.
- [Antipadrões](../05-antipatterns.md) — o que mais quebra, por serviço.
- [Erros da borda](erros.md) · [Exemplos](exemplos.md)
