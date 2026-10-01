# monitor — exemplos de relato

Chamadas do BFF pela borda. `<chave>` é a credencial de serviço do BFF, que nunca é escrita em código.
Todas as chamadas levam os mesmos cabeçalhos do relator ([endpoints](endpoints/relatos.md#cabeçalhos)).

## Criar um relato

```http
POST /v3/monitor/suporte/v1/relatos
X-Monitor-Client-Key: <chave>
X-Relator-Username: maria
X-Relator-Org: acme
X-Relator-Tenant: <tenant>
X-Request-Id: 9f1c2a
Content-Type: application/json

{"tipo":"Falha","titulo":"","contexto":{"tela":"Pedidos"},
 "mensagens":[{"texto":"Cliquei em Salvar e nada aconteceu."}],
 "diagnostico":{"formato":1,"geradoEm":"2026-09-27T02:51:47.062Z","erros":[],"rede":[],"trilha":[],"console":[],"ambiente":{}}}
```

```json
201 {"id":42,"estado":"ABERTO","criadoEm":"2026-09-27T04:00:00Z"}
```

Como o `titulo` veio vazio, ele passa a ser a 1ª linha do relato.

## "Meus chamados", com o ponto de não lido

```http
GET /v3/monitor/suporte/v1/relatos
```

```json
200 {"total":1,"limit":50,"offset":0,"rows":[{"id":42,"tipo":"FALHA","titulo":"Cliquei em Salvar e nada aconteceu.",
     "estado":"RESPONDIDO","criadoEm":"…","atualizadoEm":"…","fechadoEm":null,"naoLido":true}]}
```

## Ler a conversa e marcar como lida

```http
GET /v3/monitor/suporte/v1/relatos/42
```

```json
200 {"id":42,"tipo":"FALHA","titulo":"…","estado":"RESPONDIDO","contexto":{"tela":"Pedidos"},"naoLido":true,
     "mensagens":[{"id":1,"autor":"USUARIO","texto":"Cliquei em Salvar e nada aconteceu.","em":"…"},
                  {"id":2,"autor":"EQUIPE","texto":"Corrigido, tente de novo.","em":"…"}]}
```

```http
POST /v3/monitor/suporte/v1/relatos/42/lido      → 204
```

## Responder a um relato fechado

```http
POST /v3/monitor/suporte/v1/relatos/42/mensagens
Content-Type: application/json

{"texto":"Voltou a falhar."}
```

```json
409 {"error":"conflict","details":"relato fechado: não aceita mensagem nem novo fechamento"}
```

O usuário abre um relato novo.
