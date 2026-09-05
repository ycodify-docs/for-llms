# shell · injeção do miolo

> **Como e quando** a casca carrega o miolo de um tenant **em runtime**, e **de onde vem o build**.
> Mecanismo: **federação de módulos** (a casca é o **host**; o miolo é um **remoto**). O browser nunca
> conhece o mapa de miolos — quem resolve é o BFF. Pré: [README](README.md), [contrato-miolo](contrato-miolo.md).

## Quando

Ao o usuário **selecionar um bounded context** no menu. Trocar de BC = **desmonta** o miolo atual
(`dispose`) e **injeta** o próximo. Adicionar um tenant **não** exige rebuild da casca.

## Como (runtime)

1. A casca pede ao BFF a URL do miolo do tenant:
   `GET /tenant/{tenantId}/miolo-manifest` → devolve a **URL do ponto de entrada** do remoto.
2. A casca **registra o remoto** daquele tenant e **carrega** o módulo `mount`.
3. A casca chama `mount(el, hostContext)` (ver [contrato-miolo](contrato-miolo.md)).

## De onde vem o build do miolo

- Cada cliente tem seu **repositório de miolo**, buildado como **remoto federado** (expõe o
  [MioloModule](contrato-miolo.md#miolomodule--o-que-o-miolo-expõe)).
- O build é **publicado** num local **acessível ao browser** (host estático / CDN) — o que a casca
  carrega é a **URL do ponto de entrada** desse build.
- O **BFF mantém o mapa `tenant → URL`** do miolo. Publicar/atualizar um miolo = atualizar esse mapa;
  a **casca não é rebuildada** e o browser **nunca** carrega o mapa inteiro (só a URL do seu tenant).

```
repo do cliente ──build──▶ remoto federado (publicado num host/CDN)
                                   ▲
casca ──GET /tenant/{id}/miolo-manifest──▶ BFF (mapa tenant→URL) ──devolve a URL──┘
casca ──registra + carrega + mount(hostContext)──▶ MIOLO renderiza no bounded context
```

## Estados

- **carregando** — enquanto resolve/baixa/monta o miolo.
- **erro** — miolo indisponível (URL/ponto de entrada inacessível): a casca mostra o erro e **a sessão
  segue válida**; o usuário pode **tentar de novo** ou **trocar de BC**.

## Notas

- **Dependências comuns** (framework de UI, runtime de tema) são **compartilhadas** entre casca e miolo,
  para evitar duplicação e divergência de versão.
- A resolução `tenant → URL` é responsabilidade do **BFF** (ver [bff](bff.md)) — nunca do browser.
- Só se resolve o miolo de um tenant que **pertence ao usuário** (consta no token) — ver [seguranca](seguranca.md).
