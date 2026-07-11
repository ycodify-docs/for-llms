# orgid — guia do serviço

> **Papel:** serviço de **identidade e acesso (IAM)** da plataforma. Gerencia **organizações**,
> **contas**, **papéis (RBAC)** e **contratos**. É **pré-requisito de identidade**: a organização, a
> conta e os papéis criados aqui são o que os demais serviços (forger e o resto) exigem para autorizar
> operações. Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Contents
- Papel e posição no fluxo
- Dois domínios (`/up` e `/ua`)
- Pré-requisitos do chamador (auth + papéis)
- Conceitos próprios
- Índice de endpoints
- Ciclo de vida geral
- Pontos de coordenação
- Pitfalls (checklist do agente)

---

## Papel e posição no fluxo

orgid vem **antes** do forger: primeiro existe uma **organização** com uma **conta** dona (papel de
administrador); só então o forger implanta recursos **sob aquela organização** (`/org/{org}/…`). Os
**papéis** atribuídos aqui (ex.: administrador, engenheiro) são exatamente os que o forger e os demais
verificam para autorizar. Ver [fluxo de deploy](../03-fluxo-de-deploy.md) e [coordenação](../coordenacao.md).

> orgid **não emite token** — ele **valida** o token de acesso recebido (o login/emissão fica na borda
> da plataforma; ver [autenticação](../06-autenticacao.md)). orgid administra **quem é** e **o que pode**.

## Dois domínios (`/up` e `/ua`)

| Prefixo | Domínio | Para quê |
|---|---|---|
| `/up` | **plataforma** | organizações, contas e contratos dos **clientes/parceiros** que operam a plataforma (papéis `API_*`). |
| `/ua` | **externo** | contas e papéis dos **usuários finais** dos sistemas construídos sobre a plataforma (papéis com **espaço de nomes** por dono). |
| `/open` | **público** | endpoints **sem autenticação por design** (registro, recuperação de senha, verificação de existência). |

## Pré-requisitos do chamador (auth + papéis)

- **Autenticação:** endpoints `/up/*` e `/ua/*` exigem `Authorization` (token de acesso, esquema
  portador). Endpoints `/open/*` são **públicos** (sem auth).
- **Autorização (RBAC):** papéis da plataforma — **administrador** (`API_MASTER`), **engenheiro**
  (`API_ENGINEER`), **analista** (`API_ANALYST`), **financeiro** (`API_FINANCIAL`). A autorização é
  **escopada por organização**: o portador precisa do papel **naquela** organização. Falta de papel → `403`.
- **Sem `X-Tenant-Id`:** orgid identifica pelo **token** e pelos parâmetros de caminho
  (`{orgName}`/`{orgOwner}`); não usa o cabeçalho de tenant dos serviços de dados.

## Conceitos próprios

| Termo | Definição |
|---|---|
| **organização (org)** | Unidade de propriedade/escopo. Tem `name` (único), **`client`** (nome da empresa cliente, obrigatório na criação, ≤12), `alias`, `owner`, `status`. O `name` é o `{org}` usado nos caminhos do forger. |
| **conta (account)** | Usuário. Domínio `/up` (plataforma) ou `/ua` (externo). Tem `username`, `email`, `status`. |
| **papel (role)** | Permissão nomeada. Em `/up` são papéis fixos `API_*`; em `/ua` são papéis com **dono** (`owner`) e visibilidade (`ispublic`). |
| **associação conta-papel-org** | Vínculo tripla (conta × papel × organização) — base do multi-tenant/autorização no domínio `/up`. Em `/ua`, vínculo **conta-papel**. |
| **contrato** | Assinatura/plano de uma organização (`status`, validade, plano). |

## Índice de endpoints

> Apenas endpoints **autenticados** ou **públicos por design** são documentados. Endpoints internos
> (prefixos ofuscados de uso entre serviços), de teste e de observabilidade **não** constam.

> Organizado **por domínio** (`/up` plataforma, `/ua` externo, `/open` público). Cada endpoint tem
> método, path-variables (com significado), cabeçalhos, papéis, corpo (POST/PUT) e resposta/status.

| Domínio · Recurso | Documento |
|---|---|
| `/up` · Organização | [endpoints/up-organizacao.md](endpoints/up-organizacao.md) |
| `/up` · Conta | [endpoints/up-conta.md](endpoints/up-conta.md) |
| `/up` · Associação conta-papel-org | [endpoints/up-associacao.md](endpoints/up-associacao.md) |
| `/up` · Contrato ⚠️ *(ignorar por ora)* | [endpoints/up-contrato.md](endpoints/up-contrato.md) |
| `/ua` · Conta | [endpoints/ua-conta.md](endpoints/ua-conta.md) |
| `/ua` · Papel | [endpoints/ua-papel.md](endpoints/ua-papel.md) |
| `/ua` · Associação conta-papel | [endpoints/ua-associacao.md](endpoints/ua-associacao.md) |
| `/open` · Público (existência, registro, hash, e-mail) | [endpoints/publico.md](endpoints/publico.md) |

Erros: [erros.md](erros.md). Exemplos: [exemplos.md](exemplos.md). Contrato de fio: [openapi.yaml](openapi.yaml).

## Ciclo de vida geral

1. **Autenticação** — `/up`,`/ua`: valida o token (`Authorization`). `/open`: sem auth.
2. **Autorização** — confere o **papel** exigido **na organização** alvo. Falha → `403`.
3. **Validação** — corpo/parâmetros (campos obrigatórios, formatos). Falha → `400`.
4. **Efeito** — cria/atualiza/remove a entidade; vínculos (conta-papel-org) e contratos podem envolver
   verificação cruzada (forger/persistência — ver coordenação).
5. **Resposta** — `200`/`201`/`204`; erros conforme [erros.md](erros.md). Senha **nunca** é devolvida.

## Pontos de coordenação

- **CP-8** — ao **criar organização**, orgid consulta o forger para garantir que a org ainda não tem
  recursos (guarda de unicidade).
- **CP-9** — **contratos** são persistidos/consultados via os serviços de persistência (escrita/leitura).
- **CP-10** — orgid fornece a **organização** (`{org}`) e os **papéis** que o forger e os demais usam
  para **autorizar** (ex.: criar recurso exige papel de administrador/engenheiro **na org**).

Ver [coordenação](../coordenacao.md).

## Pitfalls (checklist do agente)

- [ ] Criar **organização + conta dona** (orgid) **antes** de usar o forger sob aquela org.
- [ ] Enviar `Authorization` em `/up`/`/ua`; `/open/*` é público (não enviar token não é erro lá).
- [ ] Autorização é **por organização** — ter o papel numa org não dá poder em outra.
- [ ] Distinguir domínios: `/up` (plataforma, papéis `API_*`) vs `/ua` (externo, papéis com dono).
- [ ] Senha nunca volta nas respostas; recuperação é via fluxo de **hash** (`/open/*`).
- [ ] `name` da organização é o `{org}` dos caminhos do forger — escolha-o cedo e mantenha.
