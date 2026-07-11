# Documentação da plataforma (para agentes)

> Documentação de API dos serviços da plataforma, destinada a **agentes de IA** que constroem sistemas
> sobre ela. Descreve **contratos** e **comportamento** (o ciclo de vida de cada requisição), de forma
> independente de tecnologia. Para um índice navegável por máquina, ver [`llms.txt`](llms.txt).

## Ordem de leitura recomendada

1. [`02-conceitos.md`](02-conceitos.md) — vocabulário e modelo mental (DDD + CQRS-ES). **Leia primeiro.**
2. [`00-plataforma.md`](00-plataforma.md) — os oito serviços e sua composição.
3. [`01-arquitetura.md`](01-arquitetura.md) — fluxo ponta a ponta entre os serviços.
4. [`03-fluxo-de-deploy.md`](03-fluxo-de-deploy.md) — como implantar um sistema do zero.
5. [`coordenacao.md`](coordenacao.md) — pontos de articulação entre serviços.

## Documentação por serviço

| Serviço | Papel | Doc |
|---|---|---|
| **auth** | Emissão de token (login): autentica e emite o token de acesso com a identidade. | [`auth/`](auth/README.md) |
| **orgid** | Identidade e acesso (IAM): organizações, contas, papéis, contratos. Pré-requisito de identidade. | [`orgid/`](orgid/README.md) |
| **forger** | Implanta os recursos do sistema (sob a organização). | [`forger/`](forger/README.md) |
| **persistence-crs** | Executa comandos de agregados (escrita). | [`persistence-crs/`](persistence-crs/README.md) |
| **es-n** | Despacha eventos para projeção e coordenação. | [`es-n/`](es-n/README.md) |
| **persistence-q** | Consulta projeções (leitura). | [`persistence-q/`](persistence-q/README.md) |
| **br-service** | Regras de negócio e coordenação de agregados. | [`br-service/`](br-service/README.md) |
| **filer** | Arquivos anexos a agregados (upload/download/listar/remover). | [`filer/`](filer/README.md) |

> **Camada de aplicação (não é um serviço da plataforma):** a **casca (shell) universal + BFF** — a
> aplicação única que loga o usuário, descobre suas permissões pelo token e **injeta o miolo** por tenant.
> Consome os oito serviços via BFF. Doc: [`shell/`](shell/README.md).

## Por tarefa

| Quero… | Comece por |
|---|---|
| Entender o vocabulário e o modelo mental | [`02-conceitos.md`](02-conceitos.md) |
| Autenticar e obter um token (login) | [`auth/`](auth/README.md) + [`auth/endpoints/sign-in.md`](auth/endpoints/sign-in.md) |
| Criar organização/conta/papéis (identidade, antes do forger) | [`orgid/`](orgid/README.md) + [`orgid/endpoints/publico.md`](orgid/endpoints/publico.md) |
| Entender o fluxo ponta a ponta entre serviços | [`01-arquitetura.md`](01-arquitetura.md) |
| Implantar um sistema do zero (ordem dos recursos) | [`03-fluxo-de-deploy.md`](03-fluxo-de-deploy.md) + [`forger/`](forger/README.md) |
| Escrever um modelo de domínio (`.model.json`) | [`examples/`](examples/README.md) + [`forger/endpoints/model.md`](forger/endpoints/model.md) |
| Criar as projeções (tabelas de leitura) | [`forger/endpoints/entity.md`](forger/endpoints/entity.md) |
| Executar/disparar um comando de domínio | [`persistence-crs/endpoints/comando.md`](persistence-crs/endpoints/comando.md) |
| Ler o estado ou o histórico de um agregado | [`persistence-crs/endpoints/agregado-leitura.md`](persistence-crs/endpoints/agregado-leitura.md) |
| Consultar projeções (query) | [`persistence-q/endpoints/consulta.md`](persistence-q/endpoints/consulta.md) |
| Escrever uma regra de negócio ou coordenação | [`br-service/README.md`](br-service/README.md) |
| Entender quando/como os serviços se chamam | [`coordenacao.md`](coordenacao.md) |
| Entender o despacho de eventos (projeção, saga) | [`es-n/README.md`](es-n/README.md) |
| Anexar/baixar arquivos de um agregado | [`filer/`](filer/README.md) |
| Gerar código a partir do contrato de fio (OpenAPI) | `<serviço>/openapi.yaml` (auth, orgid, forger, persistence-crs, persistence-q, br-service, filer) |
| Construir a aplicação do cliente (casca universal + BFF) | [`shell/`](shell/README.md) |
| Injetar o miolo de um tenant em runtime | [`shell/injecao.md`](shell/injecao.md) |
| Entender o contrato casca↔miolo | [`shell/contrato-miolo.md`](shell/contrato-miolo.md) |
| Evitar erros comuns | [`05-antipatterns.md`](05-antipatterns.md) |
| Entender o isolamento multi-tenant dos agentes + entrega de código | [`07-isolamento-e-entrega.md`](07-isolamento-e-entrega.md) |

## Exemplos concretos

[`examples/`](examples/README.md) — modelos de domínio reais (genericizados) que exercitam forger,
persistence-crs, es-n e br-service ponta a ponta.

## Convenções

- A documentação é **independente de tecnologia**: componentes de infraestrutura aparecem por papéis
  abstratos (cache distribuído, broker de mensagens, banco relacional, registro de serviços). Ver a
  tabela de vocabulário em [`02-conceitos.md`](02-conceitos.md).
- Cada serviço segue o **mesmo esqueleto**: `README.md` (guia + ciclo de vida), `endpoints/` (um por
  endpoint), `erros.md` (catálogo) e `exemplos.md`.
- Apenas endpoints autenticados e expostos ao cliente são documentados.

---

> **Versão da documentação:** 1.0 · **Última revisão:** 2026-06-20 · Histórico: [CHANGELOG.md](CHANGELOG.md).
> Índice navegável por máquina: [`llms.txt`](llms.txt); corpus completo num arquivo: [`llms-full.txt`](llms-full.txt).
