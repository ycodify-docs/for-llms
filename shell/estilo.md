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

- Existe **um** primitivo: `--yc-brand` (o verde da Ycodify). Os tokens de **acento** derivam dele.
- Ao entrar num tenant, o cliente pode **sobrescrever** `--yc-brand` (cor vinda da configuração do tenant)
  e **toda a casca recolore** — sem recompilar, sem trocar componentes.
- Voltar ao padrão = remover a sobrescrita.

## Esquemas de cor nomeados (paleta completa, por organização)

- Além do primitivo de marca, a casca oferece **esquemas de cor nomeados** — paletas completas
  (marca + superfícies + texto + acento) empacotadas sob um **id** (ex.: `verde` (padrão), `azul`,
  `ambar`, `roxo`, `cinza`). O catálogo é a **fonte única** no shell; o BFF **valida** contra o mesmo conjunto.
- Aplicação em runtime: um **atributo no elemento raiz** (`data-scheme` no `<html>`), no mesmo espírito
  da classe de tema — o CSS resolve, **sem re-render**; o **miolo herda** e recolore junto. Padrão
  (`verde`) = **sem atributo** (tokens base).
- Cada esquema define os tokens para **claro e escuro** (blocos mutuamente exclusivos). Os tokens de
  **estado** (`ok/warn/info/danger`) permanecem semânticos — **não** mudam por esquema.
- **Escopo = organização**: o esquema é uma preferência **org-scoped**, persistida via o canal de
  preferências da org (ver [bff](bff.md)) e aplicada a **todos** os usuários da org. Só **MASTER-na-org**
  altera (revalidado no servidor).

## Grupos de token (papéis)

| Grupo | Exemplos | Uso |
|---|---|---|
| superfícies | `bg`, `surface`, `surface-2` | fundos de tela/painel/hover |
| bordas | `border`, `border-2` | separadores, contornos |
| texto | `text`, `text-2`, `text-3` | primário / secundário / terciário |
| acento (deriva da marca) | `accent`, `accent-fg`, `accent-weak`, `accent-ink` | ação primária, realce, texto legível sobre claro |
| estados | `ok`, `warn`, `info`, `danger` (+ `-weak`) | status, alertas |

## Chrome da casca (o que é fixo)

- **Barra superior**: marca, **seletor de organização** (organizações do token), busca/paleta de comando,
  **tema** (claro/escuro), notificações, **perfil**, encerrar sessão.
- **Menu**: **1 entrada por bounded context** da organização ativa; estado ativo destacado com o acento.
- **Estados da área central**: **carregando** / **erro** / **pronto** (o miolo montado). "Vazio" é do
  **miolo** (conteúdo), não da casca.
- **Tipografia**: uma família de texto e uma **monoespaçada** (identificadores técnicos: tenant, papéis).

## Regras a respeitar (checklist)

- [ ] **Nunca** cravar cor/medida no componente — usar o **token semântico**.
- [ ] **Miolo herda os tokens da casca** — o mesmo tema/marca vale para casca e miolo (coerência visual).
- [ ] Suportar **claro e escuro** — todo componente novo funciona nas duas camadas.
- [ ] **Marca só via `--yc-brand`** — não referenciar o verde direto; usar `accent*`.
- [ ] **Identificadores técnicos em monoespaçada** (tenant/papel/estado).
