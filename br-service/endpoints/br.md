# br-service · endpoints · executar função (rota)

> Executa o **processador** identificado por uma **rota**. Chamado **pelo persistence-crs**, não pelo
> cliente. Guia: [../README.md](../README.md).

## Requisição

`POST /br`

Corpo:

```json
{ "route": "<rota do processador>", "data": { "...": "..." } }
```

- `route` (obrigatório) — caminho textual que identifica o processador, na forma canônica, **começando pela organização** (ex.: `acme/vendas/vendas/pedido/validar`).
- `data` — dados passados ao processador. Se ausente, o corpo inteiro é passado.

## Resposta

- `200` — o objeto retornado **diretamente** pelo processador (sem envelope):

```json
{ "...": "..." }
```

- `400` — rota inexistente ou exceção no processador:

```json
{ "status": "error", "mensagem": "<descrição>", "tipo": "<tipo do erro>" }
```

- `400` — **falha de validação do corpo** (corpo nulo, não-objeto, ou sem `route`). Corpo **diferente**,
  com um campo só:

```json
{ "erro": "<descrição>" }
```

## Comportamento
Segue o [ciclo de vida](../README.md#ciclo-de-vida-de-uma-requisição): validação → roteamento →
execução do processador → resposta direta. O serviço é um **transformador puro** (sem efeitos colaterais
externos próprios).

## Erros
`400` em todos os casos de falha (validação, rota desconhecida — com lista de rotas disponíveis —,
exceção do processador), mas em **dois formatos de corpo diferentes**: a falha de validação devolve
`{ erro }` e todo o resto devolve `{ status, mensagem, tipo }`. Catálogo: [../erros.md](../erros.md).
