# Princípios de design da Shell Universal

> **Fundamentos de construção** — Ycodify · Shell Universal
> Guia de identidade e governança visual · Direção Atlas · v1.0 · jul/2026
> Documento vivo — revisar a cada release maior.

Este documento define os **valores, princípios e estruturas** que orientam a construção da casca de aplicação da Ycodify — a moldura única que hospeda qualquer produto, para qualquer organização (tenant), preservando uma só identidade visual.

Ele é a **fonte da verdade**. À medida que a aplicação evolui — novos módulos, novas telas, novos tenants — as decisões aqui registradas devem ser respeitadas para manter uniformidade e coerência. Divergências são permitidas apenas quando promovidas a nova regra neste guia.

---

## 01 — Filosofia

### A casca se molda ao tenant, não o contrário

A Shell é uma estrutura estável e reconhecível dentro da qual o conteúdo varia. O usuário entra uma vez e navega entre organizações, projetos e módulos sem reaprender a interface. A identidade da Ycodify permanece constante; o que muda é o *dado*, o *papel* do usuário e, no limite, a *cor de marca* do tenant.

Toda decisão de design responde a uma pergunta: **isso ajuda a casca a acomodar o desconhecido sem perder a própria identidade?**

---

## 02 — Valores

### Cinco compromissos inegociáveis

| # | Valor | Compromisso |
|---|-------|-------------|
| **V1** | Familiaridade acima de novidade | Um padrão previsível vale mais que um efeito inédito. Reuso de componente antes de invenção. |
| **V2** | Clareza acima de densidade | Ar, hierarquia e cantos macios. Nunca encher a tela porque há espaço — cada elemento deve justificar sua presença. |
| **V3** | Contexto sempre visível | O usuário nunca se pergunta em qual organização, papel ou tela está. A barra superior responde antes da dúvida. |
| **V4** | Um token, um sistema | Nenhuma cor, espaçamento ou raio literal no código de tela. Tudo deriva dos tokens semânticos — a única via de mudança. |
| **V5** | Todo estado é projetado | Pronto, vazio, carregando e erro não são acidentes — são quatro telas desenhadas com o mesmo cuidado. Uma feature só está pronta quando os quatro existem. |

---

## 03 — Identidade visual

### Marca e cor

O verde Ycodify é o único primitivo de marca. Todos os tons de acento derivam dele. O tenant pode sobrescrever `--yc-brand` em runtime e a casca inteira recolore — sem recompilar, sem quebrar contraste.

| Token | Hex | Uso |
|-------|-----|-----|
| `--yc-brand` | `#00d676` | Verde de marca (sobrescrevível pelo tenant) |
| `--accent-ink` | `#07935a` | Texto de acento sobre fundo claro |
| `--accent-weak` | `#dcf7e9` | Fundo suave de acento |
| `--accent-fg` | `#043019` | Texto sobre o verde cheio |

> **Regra:** o verde cheio nunca recebe texto que não seja `--accent-fg`. Para texto de acento sobre fundo claro, use sempre `--accent-ink`, nunca o verde puro (contraste insuficiente).

#### Cores de estado (semânticas, não decorativas)

| Estado | Token | Hex |
|--------|-------|-----|
| Sucesso | `--ok` | `#1f7a4d` |
| Atenção | `--warn` | `#9a6412` |
| Informação | `--info` | `#1f6f9a` |
| Erro | `--danger` | `#b23b34` |

### Tipografia

- **Manrope** (400 · 500 · 600 · 700 · 800) — interface inteira: títulos, corpo, rótulos, botões. Peso 800 para títulos, 700 para ênfase, 500–600 para corpo e controles.
- **IBM Plex Mono** (400 · 500 · 600) — metadados técnicos: identificadores, códigos de org, atalhos, timestamps, rótulos em caixa-alta. Nunca para corpo de leitura.

> Duas famílias, sem exceção. Uma terceira fonte só entra por decisão registrada neste guia. Corpo mínimo de interface: 13px; texto de leitura longa: 14–15px com entrelinha 1.6.

### Espaço, raio e elevação

| Dimensão | Regra | Valores |
|----------|-------|---------|
| Espaçamento | Grade de 4px; sempre múltiplos | 6 · 10 · 12 · 18 · 24 |
| Raio | Macio por padrão; pílula p/ ações e chips | 10 · 14 controles · 999 pílula |
| Altura de controle | Toque confortável; alvo mín. 38px | 38 · 42 (perfil) |
| Bordas | 1px, tom de borda semântico | `--border` · `--border-2` |
| Elevação | Uma sombra só (popovers/menus); nunca empilhar | `--shadow` |

> Sombra é sinal de camada flutuante, não de decoração. Superfícies base (cards, painéis) usam borda, não sombra.

---

## 04 — Estrutura

### Anatomia da casca

A moldura é fixa. O conteúdo é o que respira dentro dela.

- **Barra superior — sempre presente:** marca · seletor de organização · busca/comando (⌘K) · tema · notificações · perfil.
- **Navegação lateral:** reconstruída a partir dos papéis org-scoped do token. O menu é do usuário, não fixo.
- **Área de conteúdo:** o único espaço que varia por módulo. Herda todos os tokens; nunca introduz cor ou fonte própria.

> O menu não é uma constante desenhada: é derivado dos papéis que o token do usuário carrega naquela organização. Ao trocar de org, a navegação se reconstrói.

### Os quatro estados de toda tela

- **Pronto** — o caso feliz. Dados carregados, ações disponíveis. Densidade equilibrada, sem sobrecarga.
- **Vazio** — nunca uma tela em branco. Explica o que aparecerá ali e oferece a primeira ação.
- **Carregando** — esqueleto com a forma do conteúdo real (shimmer), não um spinner solto. Preserva o layout.
- **Erro** — diz o que falhou em linguagem humana e oferece um caminho de recuperação. Nunca um código cru.

---

## 05 — Governança

### Regras para evoluir sem perder identidade

**✓ Faça**

- Derive toda cor, espaço e raio dos tokens semânticos.
- Projete os quatro estados antes de considerar a feature pronta.
- Reuse um componente existente antes de criar um novo.
- Teste claro e escuro em toda tela nova.
- Promova qualquer padrão novo a regra neste guia.

**✕ Não faça**

- Escrever cor/medida literal (hex, px de cor) na tela.
- Introduzir uma terceira fonte ou nova família de cor.
- Empilhar sombras ou usar sombra como decoração.
- Entregar uma tela em branco como estado vazio.
- Alterar a moldura (barra, navegação) por módulo.

### O teste de identidade

Coloque a tela nova ao lado de qualquer tela existente. Se um usuário não conseguir dizer que pertencem ao mesmo produto — sem olhar o logo — a tela nova ainda não está pronta. **Uniformidade é o recurso, não o acabamento.**

---

*Este é um documento vivo. Toda decisão que amplia ou altera o sistema deve ser refletida aqui na mesma entrega — o guia e o produto nunca divergem. Referência técnica de tokens: `react-reference/theme.css`.*
