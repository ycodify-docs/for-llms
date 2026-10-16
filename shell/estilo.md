# shell · guia-mestre de estilo

> As regras de estilo **a respeitar** na casca e nos miolos. O princípio: **tokens semânticos** definidos
> como **variáveis** e consumidos por utilitários — nunca cores/medidas "cravadas" no componente. Um único
> primitivo, a **marca**, é **sobrescrevível pelo tenant** em runtime: a casca inteira recolore.
> Pré: [README](README.md). _(Doc de estilo — necessariamente concreta; os **contratos** ficam em
> [contrato-miolo](contrato-miolo.md)/[bff](bff.md).)_

## Princípio: tokens semânticos, não valores cravados

- Cada cor/sombra é uma **variável semântica** (`--surface`, `--text`, `--accent`, `--border`, …), não um
  hex no componente. O componente usa o **nome do papel** (ex.: fundo = `surface`, texto = `text`),
  nunca "#fff".
- Há **duas camadas**: **clara** (padrão) e **escura**. A troca é por **classe no elemento raiz** — o CSS
  resolve o resto, **sem** re-render por tema.
- A preferência de tema é **persistida por usuário**.

## Marca sobrescrevível (rebranding por tenant)

- Existe **um** primitivo: `--yc-brand` (o verde da Ycodify, no esquema padrão). Os tokens de **acento**
  derivam dele.
- A cor de cada empresa vem do **esquema** do manifesto da aplicação (abaixo): o esquema redefine
  `--yc-brand` e os tokens que dependem dela, e **toda a casca recolore** — sem recompilar, sem trocar
  componentes. Não há outra fonte de cor por cliente: a sobrescrita avulsa de `--yc-brand` existe no código
  da casca, mas nada a alimenta.
- Voltar ao padrão = esquema `verde` (sem atributo).

## Esquemas de cor nomeados (paleta completa, por organização)

- Além do primitivo de marca, a casca oferece **esquemas de cor nomeados** — paletas completas
  (marca + superfícies + texto + acento) empacotadas sob um **id**: `verde` (padrão), `azul`, `ambar`,
  `roxo`, `cinza` e `marinho`. O catálogo é **contrato** entre a casca, o BFF e a lista fechada do orgid:
  mudar um é mudar os três.
- **`marinho`** é de **duas cores**: azul-marinho dominante e acento verde-lima (criado em 2026-10-16). Nele o
  texto sobre o acento é o próprio marinho.
- Aplicação em runtime: um **atributo no elemento raiz** (`data-scheme` no `<html>`), no mesmo espírito
  da classe de tema — o CSS resolve, **sem re-render**; o **miolo herda** e recolore junto. Padrão
  (`verde`) = **sem atributo** (tokens base).
- Cada esquema define os tokens para **claro e escuro** (blocos mutuamente exclusivos). Os tokens de
  **estado** (`ok/warn/info/danger`) permanecem semânticos — **não** mudam por esquema.
- **Quem escolhe é a empresa, no manifesto da aplicação** da org ([orgid](../orgid/endpoints/aplicacao.md)),
  e vale para todos os usuários dela. Antes do login, vem do **domínio**; depois, da **org ativa**
  ([bff — identidade](../bff/README.md#identidade-da-aplicação-appidentidade)). **Não há seletor de cor na
  casca** e o esquema não é mais preferência da org (desde 2026-10-16).

## Grupos de token (papéis)

| Grupo | Exemplos | Uso |
|---|---|---|
| superfícies | `bg`, `surface`, `surface-2`, `sidebar` | fundos de tela/painel/hover; `sidebar` é o menu lateral |
| bordas | `border`, `border-2` | separadores, contornos |
| texto | `text`, `text-2`, `text-3` | primário / secundário / terciário |
| acento (deriva da marca) | `accent`, `accent-fg`, `accent-weak`, `accent-ink` | ação primária, realce, texto legível sobre claro |
| dominante | `dominante` | a cor que identifica a empresa em área grande (ex.: a metade colorida do cartão de login). É o próprio `accent`, salvo em esquema de duas cores, como o `marinho` |
| tinta | `ink`, `ink-fg` | botão escuro de ação secundária forte (ex.: relatar ao suporte); inverte no escuro |
| estados | `ok`, `warn`, `info`, `danger` (+ `-weak`; `danger-line` para borda) | status, alertas, campo com erro |
| pílula de estado | `pill-bg-l`, `pill-fg-l` | só a **luminosidade** por tema; ver abaixo |

## Pílula de estado do agregado

- Cada estado tem **uma matiz**, estável entre telas: a pílula usa
  `oklch(var(--pill-bg-l) <croma> <matiz>)` no fundo e `oklch(var(--pill-fg-l) <croma> <matiz>)` no texto.
  O tema troca só a luminosidade; a matiz é a mesma no claro e no escuro.
- A matiz **não se escreve por cliente no código**: sai do nome do estado por uma função estável — o
  mesmo nome dá sempre a mesma cor, em qualquer tela e para qualquer tenant.
- Texto da pílula em monoespaçada.

## Chrome da casca (o que é fixo)

- **Telas públicas** (login, criar conta, esqueci a senha, código do e-mail): um cartão dividido, com a
  metade esquerda inteira na cor `dominante` e o formulário à direita, encabeçado pelo **ícone e pelo nome
  da empresa**; em tela estreita, só o formulário. No rodapé, os links de **termos** e **política de
  privacidade** (texto simples, em modal); "Criar conta" só quando a identidade tem `autocadastro`, com
  aceite obrigatório por caixa marcada.
- **Barra superior** (64px): **ícone e nome da empresa** (da identidade da aplicação), **seletor de organização** (organizações do token), busca/paleta de
  comando, agente, **tema** (claro/escuro), **suporte**, notificações e o **menu da conta** — é nele, e só
  nele, que fica **encerrar sessão**.
- **Menu lateral**: **Projeto → Contexto → Agregados** da organização ativa. Continua sendo **1 entrada
  por bounded context**, agora com os agregados dela como filhos, cada um com a contagem de registros; o
  agregado ativo fica destacado com o acento. O agregado se escolhe **aqui**, e a escolha remonta o miolo
  (ver [contrato-miolo](contrato-miolo.md)). No rodapé: suporte, a chave do tenant (clicar copia) e o
  status do miolo, medido por health check.
- **Larguras**: abaixo de 1000px o menu lateral vira gaveta; a barra superior encolhe por faixas (nome do
  usuário some abaixo de 1100px, a busca vira ícone abaixo de 860px, agente e tema somem abaixo de 720px).
  A página nunca rola na horizontal.
- **Estados da área central**: **vazio** (nenhum agregado escolhido), **carregando** (esqueleto),
  **erro** e **pronto** (o miolo montado). O vazio **de conteúdo** — agregado sem registros — é do miolo.
- **Tipografia**: uma família de texto e uma **monoespaçada** (identificadores técnicos: tenant, papéis).

## Regras a respeitar (checklist)

- [ ] **Nunca** cravar cor/medida no componente — usar o **token semântico**.
- [ ] **Miolo herda os tokens da casca** — o mesmo tema/marca vale para casca e miolo (coerência visual).
- [ ] Suportar **claro e escuro** — todo componente novo funciona nas duas camadas.
- [ ] **Marca só via `--yc-brand`** — não referenciar o verde direto; usar `accent*`.
- [ ] **Identificadores técnicos em monoespaçada** (tenant/papel/estado).
