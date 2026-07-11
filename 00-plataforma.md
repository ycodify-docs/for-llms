# A plataforma — composição

> Visão de alto nível: os oito serviços, seus papéis e como compõem um sistema cliente.
> Pré-requisito de leitura: [conceitos](02-conceitos.md).

## O que a plataforma faz

A plataforma transforma **modelos declarativos** num sistema operante. Um cliente descreve seu domínio
(contextos, agregados, comandos, eventos, projeções) e a plataforma **interpreta** esses modelos: cria
os recursos de infraestrutura, executa comandos, mantém projeções e responde consultas — sem que o
cliente escreva um serviço de backend próprio.

Os serviços formam um **time de interpretadores de modelos**. Um deles (**forger**) implanta os recursos;
os demais operam sobre eles em tempo de execução.

## Os oito serviços

| Serviço | Papel | Como é usado |
|---|---|---|
| **auth** | **Emissão de token (login)**: autentica credenciais e emite o token de acesso com a identidade (usuário, papéis, organizações, tenants). | Porta de entrada: faça login e use o token em `Authorization` nos demais. Emite; não valida. |
| **orgid** | **Identidade e acesso (IAM)**: organizações, contas, papéis (RBAC) e contratos. **Pré-requisito de identidade.** | Cria a organização e a conta dona (com papéis) que os demais serviços exigem para autorizar. Valida o token (não o emite). |
| **forger** | Implanta **todos os recursos** necessários para que os demais serviços operem. É o primeiro a **implantar recursos** (após a identidade existir). | Define metadados de conexão, bancos, contextos, esquemas, projeções e publica os modelos de domínio e de processo. |
| **persistence-crs** | Executa os **comandos** definidos para um agregado num bounded context (lado de escrita do CQRS-ES). | Recebe um comando, valida a transição de estado, eventualmente aplica regra/coordenação, grava o evento. |
| **es-n** | Daemon que **escuta mudanças de estado** e **despacha** o agregado para filas. | A cada evento, reconstitui o agregado em JSON e o envia para atualização de projeção e/ou coordenação. |
| **persistence-q** | Permite **consultar** o estado dos agregados pela ótica de suas **projeções** (lado de leitura). | Recebe um critério de consulta e devolve as projeções materializadas correspondentes. |
| **br-service** | Executa **regras de negócio** e funções de **coordenação** de agregados (comumente em sagas). | É provocado **somente** pelo persistence-crs, quando o modelo do comando assim determina. |
| **filer** | Serviço **transversal de arquivos**: guarda/serve arquivos **anexos a agregados**. | Upload/download/listar/remover, identificando o arquivo por `org/project/entity/entityId/attribute`. |

## Como os serviços se compõem (resumo)

0. **orgid** estabelece a **identidade**: organização + conta dona + papéis. **auth** emite o **token**
   (login) com essa identidade. Nada de recurso é criado antes de existir a org (o `{org}`), a
   autorização e um token válido.
1. **forger** prepara o terreno: cria os recursos e publica os modelos, **sob a organização** do orgid.
2. **persistence-crs** passa a aceitar comandos para os agregados descritos no modelo publicado.
3. Cada comando bem-sucedido grava um evento. A mudança de estado **notifica** o **es-n**.
4. **es-n** reconstitui o agregado e o **despacha** para filas: o caminho comum atualiza a **projeção**
   no banco de leitura; outros caminhos conduzem à **coordenação** entre agregados/contextos.
5. **persistence-q** consulta as projeções mantidas por esse fluxo.
6. **br-service** entra quando um comando exige regra de negócio ou coordenação — sempre acionado pelo
   persistence-crs.

O detalhamento do fluxo ponta-a-ponta está em [arquitetura](01-arquitetura.md). A ordem obrigatória de
implantação está em [fluxo de deploy](03-fluxo-de-deploy.md). Os pontos de articulação entre serviços
estão consolidados em [coordenação](coordenacao.md).
