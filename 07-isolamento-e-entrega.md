# 07 · Isolamento dos agentes + entrega de código

Como a plataforma **isola** o trabalho dos agentes por cliente e como o **código** que eles produzem chega ao ar.
Linguagem direta, com diagramas. (Decisão registrada em `se-agents/docs/adr/0007`; itens de infra em
`HANDOFF_YCOGNIO_III`.)

> **Modelo conceitual** — caminhos absolutos, credenciais e regras de rede concretas ficam na configuração de
> deploy (fora deste documento). Aqui está o **desenho**, não os segredos.

---

## 1. O problema

A hierarquia de agentes escreve arquivos (docs, `.model.json`) e, às vezes, código (frontend, microfrontends,
processors) **por cliente**. Sem uma fronteira dura, um agente poderia alcançar **outro cliente** ou o resto do
servidor. Como é **multi-tenant**, isso é o risco principal. A regra tem de ser **à prova de instrução**: nem uma
ordem do usuário fura o isolamento (salvo exceção documentada aqui).

## 2. A "sala trancada" — uma por cliente

Cada sistema de cliente ganha uma **pasta-raiz** (um diretório por cliente/sistema, definido no deploy). O agente
trabalha **dentro** dessa raiz; ali só existe o material daquele cliente.

```
   Cliente A          Cliente B          Cliente C
   ┌────────┐         ┌────────┐         ┌────────┐
   │ sala A │         │ sala B │         │ sala C │
   └────────┘         └────────┘         └────────┘
   A nunca vê B. Se o agente de A for comprometido, ele
   não alcança B — a sala de B não existe no mundo dele.
```

## 3. Três camadas de proteção (da mais forte à mais leve)

```
   ┌── 3. CONVENÇÃO (o agente SABE a regra) ─────────────────┐
   │  ┌── 2. PORTEIRO (confere cada acesso) ───────────────┐ │
   │  │  ┌── 1. SALA TRANCADA (isolamento de sistema) ───┐ │ │
   │  │  │  Dentro só existe a pasta do cliente.         │ │ │
   │  │  │  O resto NÃO EXISTE para ele.                 │ │ │
   │  │  └────────────────────────────────────────────────┘ │ │
   │  └──────────────────────────────────────────────────────┘ │
   └────────────────────────────────────────────────────────────┘
```

| Camada | O que é | Força |
|---|---|---|
| **1 — Sala trancada** | um compartimento (container) que só contém a pasta daquele cliente; o proibido **não existe** — é inalcançável | a mais forte; barra até ordem do usuário |
| **2 — Porteiro** | verificação automática a cada acesso (arquivo **e** rede): é da sala/allowlist? passa; senão, **barra e explica** | reforço + erro claro |
| **3 — Convenção** | regra escrita que o agente acata; usuário **não é atendido** ao pedir acesso fora da raiz, salvo exceção documentada | não depende só de barreira |

A camada 1 é o isolamento **real** — dispensa ficar bloqueando comandos, porque o proibido simplesmente não está lá
(fecha, por construção, os escapes por caminho relativo, atalho de link, ou shell).

## 4. A janela — o que a sala alcança para fora

A sala é fechada, com uma **janela estreita** que abre **só** para 4 destinos autorizados:

```
   ┌── SALA (pasta do cliente: ler + gravar) ───────────────────┐
   │  ╔═══ JANELA — só 4 destinos ════════════════════════════╗ │
   │  ║  → 📖 documentação (repo público)          só LER     ║ │
   │  ║  → 🏭 serviços da plataforma                usar       ║ │
   │  ║  → 🧠 API do modelo de IA                   usar       ║ │
   │  ║  → 🗄️ git dos repos de código DESTE cliente enviar     ║ │
   │  ╚══════════════════════════════════════════════════════════╝│
   │  A janela NÃO abre pro resto da internet, nem p/ outro       │
   │  cliente, nem pro resto do servidor.                         │
   └──────────────────────────────────────────────────────────────┘
```

A **documentação** é consultada **pela janela** (um repo público na internet), não uma cópia dentro da sala:
versão única para todos, sempre atual, o agente **nunca a altera**. O porteiro também guarda a janela — só passa
para os 4 destinos.

## 5. O que os agentes produzem e para onde vai

**Regra de ouro**: tudo que o agente escreve, escreve **dentro da sala**. O que sai, sai por **git**, pela janela,
para destino autorizado — e **revisado por humano**.

| O agente entrega | Precisa de repo? | Destino |
|---|---|---|
| `.model.json` + esquema de tela (**o comum**) | **NÃO** — fica na sala | plataforma → a shell desenha |
| microfrontend sob medida (**raro**) | **SIM** — repo + build | armazenamento estático → a shell encaixa |
| processors (regras/coordenação) | **SIM** — repo/artefato | injetado no br-service |

O grosso do frontend **não passa por repo** — passa pelo **modelo**, que já mora na sala e já tem caminho de deploy.

## 6. Frontend — uma shell para todos, dois modos

Uma **única casca (shell)**, um **único endereço** que todo cliente acessa. O que muda por cliente é o que a casca
carrega dentro:

```
   Cliente A ─┐
   Cliente B ─┼──▶ [ 1 endereço ] ──▶ 🐚 SHELL (uma, para todos)
   Cliente C ─┘                          │  (o BFF decide por cliente:)
             ┌───────────────────────────┴───────────────────────────┐
             ▼                                                        ▼
   GEN (default, comum)                                  CUSTOM (raro, exceção)
   lê o .model.json e DESENHA a tela                     busca o BUILD do microfrontend
   na hora (sem repo, sem build)                         do cliente e ENCAIXA (repo + build)
```

O **BFF é o dono do mapa** `cliente → miolo`: sem custom, devolve o renderizador **GEN** (default); com custom,
devolve a URL do microfrontend daquele cliente. **O navegador nunca vê o mapa** (um cliente não descobre o do
outro). **GEN é o padrão; CUSTOM só com aprovação humana** (exige justificativa técnica).

## 7. O caminho do código (só CUSTOM e processors)

```
   1. agente escreve o código          → DENTRO da sala (cópia de trabalho / clone)
                 │ git → BRANCH / PR
                 ▼
   2. 👤 humano revisa e aprova         ← "humano decide; o agente não publica sozinho"
                 │
                 ▼
   3. 🤖 automação (CI) faz o BUILD     → FORA da sala (a sala não tem ferramenta de build)
                 │ publica (caminho imutável por versão)
                 ▼
   4. 📦 armazenamento estático         →  a shell (via BFF) passa a encaixar o build novo
                                          (processors: injetados no br-service)
```

Três travas de segurança no "enviar código":
- **credencial estreita** — só serve para os repos **daquele** cliente (se vazar, não toca outro);
- **credencial temporária** — só existe enquanto a sala existe;
- **só branch/PR** — o agente **propõe**; um humano **aprova** antes de virar oficial.

A sala **nunca** faz build nem publica — só guarda o código e empurra o PR. Build, publicação e atualização do mapa
acontecem **fora**, disparados pela **aprovação humana**.

## 8. Visão completa

```
   👤 usuário do cliente ──▶ [ 1 endereço ] ──▶ 🐚 SHELL ──pergunta──▶ 🗺️ BFF (mapa cliente→miolo)
                                                   │                        │
                                    ┌──────────────┴───────────┐   GEN│CUSTOM(aprovado)
                                    ▼                          ▼        │
                          lê .model.json e desenha    encaixa build ◀───┘
                          (GEN, o comum)              (CUSTOM, raro)
                                    ▲                          ▲ build publicado
   ┌────────────────────────────────┴──────────────────────────┴──────────────────────┐
   │  🔒 SALA TRANCADA do cliente                                                       │
   │     • agente escreve docs + modelo + (raro) código  ── tudo aqui                   │
   │     • janela: doc(ler) · plataforma · API · git-do-cliente                         │
   │     código sai por PR → 👤 aprova → 🤖 CI build → 📦 estático → 🗺️ BFF aponta       │
   └──────────────────────────────────────────────────────────────────────────────────────┘
        cada cliente = uma sala; A nunca vê B
```

## 9. Em uma frase

Cada cliente tem uma **sala trancada** onde o agente trabalha; a sala só enxerga a pasta daquele cliente e uma
**janela estreita** (documentação, plataforma, API, git-do-cliente); o **comum é o frontend nascer do modelo** (sem
repo); **código sob medida sai só por PR aprovado por humano**, é compilado **fora** da sala e servido por
**armazenamento estático**.
