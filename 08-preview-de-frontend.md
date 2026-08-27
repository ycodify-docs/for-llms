# 08 · Pré-visualização do frontend

Como o frontend que **você** constrói é visto e **testado pelo usuário** antes de qualquer publicação.
Pré: [`07-isolamento-e-entrega.md`](07-isolamento-e-entrega.md) (a sala trancada) e
[`bff/README.md`](bff/README.md) (o BFF, que é quem o app consome).

> **Modelo conceitual** — endereços, portas e caminhos absolutos são **configuração de deploy** e não
> constam aqui. O que está aqui é o **contrato**: onde pôr o código, o que ele precisa ter, e com quem ele fala.

---

## 1. O que é

Você escreve código de frontend dentro da sala. O usuário precisa **ver aquilo rodando** — clicar, navegar,
preencher formulário, passar pela tela de login que você construiu — para dizer se está certo.

A pré-visualização resolve isso: o ambiente monta o seu app e o serve num endereço próprio, que o usuário abre
**numa aba separada** do painel. É ambiente de **teste**, não de publicação: nada do que acontece ali vai ao ar.

**Quem dispara é o usuário**, por um botão no painel. Você **não** aciona a pré-visualização, não a comanda e não
precisa saber quando ela acontece. Seu contrato é só **deixar o app pronto para ser montado**.

## 2. Onde o código vive

Uma organização tem **um sistema**, e um sistema tem **uma pasta-raiz** (a sala). Dentro dela:

```
   <raiz do sistema>/
   ├── application/     ← O FRONTEND VAI AQUI. Nome fixo, não é configurável.
   ├── artifacts/
   ├── processors/
   └── ...
```

**`application/` é convenção fixa.** Não invente `app/`, `web/`, `frontend/`, nem aninhe o projeto mais fundo.
Se a pasta não existir, a pré-visualização responde ao usuário que **ainda não há app construído** — o que é
uma resposta correta, não um erro.

## 3. O que `application/` precisa ter

| Precisa | Por quê |
|---|---|
| Um **manifesto de dependências** na raiz de `application/` | é por ele que as bibliotecas são instaladas |
| Uma **rotina de build** declarada nesse manifesto | é o que o ambiente executa para montar o app |
| O build produzindo **`application/dist/`** | é essa pasta que é servida ao usuário |

O ambiente instala as dependências e executa a rotina de build. Se o build falhar, **o usuário vê o erro** —
com as últimas linhas do log — e a aba não abre. Um build que quebra é um app que não pode ser avaliado.

### Bibliotecas

Você **pode** escolher e instalar bibliotecas livremente: o registro público de pacotes é alcançável a partir da
sala (é um dos destinos autorizados da "janela" — ver [`07`](07-isolamento-e-entrega.md) §4).

Há um **depósito compartilhado** entre os sistemas: uma biblioteca já baixada por outro projeto é reaproveitada,
sem baixar de novo e sem ocupar espaço outra vez. Consequência prática: **não hesite em instalar** o que precisa —
a segunda instalação de uma biblioteca conhecida é praticamente instantânea.

### Verifique antes de entregar

A sala tem a mesma ferramenta que o ambiente de pré-visualização usa. **Rode o build você mesmo** antes de dizer
ao usuário que terminou. Um erro de importação, uma dependência esquecida ou um símbolo que não existe aparecem
em segundos — e é muito melhor você descobrir do que o usuário.

## 4. Com quem o app fala

**O app fala com o BFF, e só com o BFF.** Nunca com os serviços da plataforma diretamente.

```
   navegador ──▶ /api/...  (mesma origem do app)
                    │
                    ▼
              🔀 repasse
                    │
                    ▼
                 BFF ──▶ serviços da plataforma
                          (Authorization + tenant compostos no servidor)
```

Chame **`/api/...` na própria origem** do app — caminho relativo, sem host. O ambiente repassa ao BFF. Duas coisas
dependem disso, e as duas quebram se você chamar o BFF pelo endereço dele:

- **A sessão.** O BFF entrega a sessão num cookie que o navegador só guarda se a chamada for da mesma origem.
  Chamando por fora, o usuário loga e a sessão não persiste — a próxima requisição volta como não autenticado.
- **A permissão de origem.** O BFF só aceita chamadas das origens que conhece. A origem da pré-visualização não é
  uma delas. Pelo caminho relativo o problema não existe: para o navegador é tudo a mesma origem.

Então: `fetch('/api/session/login', …)`, não `fetch('https://<endereço-do-bff>/session/login', …)`.

O que o BFF oferece — sessão, capacidade, comando, consulta, agregado, histórico, autocadastro — está em
[`bff/README.md`](bff/README.md). **Nenhum segredo vai para o navegador**: o token vive no servidor.

## 5. Login é parte do que você constrói

O usuário do painel está autenticado como **conta de plataforma**, para *construir* o sistema. Isso não tem
relação com os usuários da **aplicação** que você está construindo.

Se o sistema tem login, **a tela de login é sua** — você a constrói, e é passando por ela que o usuário vai testar
a aplicação por dentro. O fluxo de autocadastro e ativação também é do BFF (`/ua/*`). Não presuma usuário já
logado: comece pela porta da frente, como um usuário real começaria.

## 6. Ciclo de vida (o que você precisa saber)

- A pré-visualização é **temporária**. Fica de pé enquanto está sendo usada e é recolhida sozinha depois de um
  período sem acesso. Não conte com ela como serviço permanente.
- **Cada pedido remonta o app.** O usuário sempre vê o estado atual do seu trabalho — não há cache do build
  anterior. Você não precisa avisar ninguém para "atualizar".
- **É por sistema, não por sessão de chat.** Duas conversas sobre o mesmo sistema veem a mesma pré-visualização.

## 7. O que a pré-visualização NÃO é

| Não é | Por quê |
|---|---|
| **Publicação** | nada ali vai ao ar. O caminho de publicação continua sendo o de [`07`](07-isolamento-e-entrega.md) §7: proposta → revisão humana → build fora → armazenamento estático. |
| **Ambiente do usuário final** | roda na máquina de quem está construindo, alcançável só dali. |
| **Coisa que você aciona** | quem decide ver é o usuário. Você deixa pronto; ele olha quando quiser. |
| **Substituto do modelo** | o caminho comum do frontend continua sendo o `.model.json` + a casca ([`07`](07-isolamento-e-entrega.md) §6). Construir app próprio é a exceção, não o padrão. |

## 8. Antipadrões

| ❌ | ✅ |
|---|---|
| Pôr o app em `app/`, `web/` ou fora da raiz | `application/`, na raiz do sistema |
| Chamar o BFF pelo endereço dele | caminho relativo `/api/...` |
| Chamar a plataforma direto do navegador | sempre pelo BFF |
| Guardar token/credencial no navegador | o BFF guarda no servidor; o app não vê |
| Cravar endereço de serviço no código | endereço é configuração de deploy |
| Entregar sem rodar o build | rode o build na sala; o erro aparece em segundos |
| Supor o usuário já autenticado | construa e exercite a tela de login |
| Pedir ao usuário para "subir o preview" | o botão é dele; você só deixa pronto |

## 9. Em uma frase

Escreva o app em **`application/`**, garanta que ele **monta** e que fala com a plataforma **só pelo BFF em
caminho relativo** — o resto (montar, servir, abrir, recolher) é do ambiente, e quem decide olhar é o usuário.
