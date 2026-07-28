# cache — guia do serviço

> **Papel:** serviço de **cache distribuído** (redis-backed) da plataforma. Guarda o **modelo publicado** de cada
> tenant (chaves de **write-model** e **read-model**) que os serviços de execução leem em runtime. É **interno** —
> **atrás do gateway, sem rota/endpoint público**; acessado apenas pela própria plataforma (não pelo cliente).
> Pré-requisitos: [conceitos](../02-conceitos.md), [arquitetura](../01-arquitetura.md).

## Papel no fluxo

- O **[forger](../forger/README.md)**, ao **publicar** um `.model.json`
  (`POST /v3/forger/org/{org}/project/{project}/tenant/{tenantId}/model`), **grava o modelo no cache** (chave do
  write-model). Criar dataschemas/entidades (projeções dos agregados) também é via forger.
- **[persistence-crs](../persistence-crs/README.md)** e **[persistence-q](../persistence-q/README.md)** **leem** o
  modelo do cache a cada request (spec por-instância, com TTL e self-heal) — detalhe do ciclo em
  [persistence-crs — carga da spec do tenant](../persistence-crs/README.md#carga-da-spec-do-tenant-cache-por-instância).

## Acesso

- **Sem endpoint público**: não é chamável via gateway (não há prefixo `/v3/cache`). Os serviços o acessam
  **internamente** (referido como `../cache` — **não** Redis direto).
- **Invalidação do modelo**: alterar um modelo em runtime exige **republicá-lo** (forger) **+ invalidar as duas
  chaves** (read-model e write-model); reiniciar o serviço sozinho não basta — ver a seção de refresh em
  [persistence-crs](../persistence-crs/README.md#carga-da-spec-do-tenant-cache-por-instância).
