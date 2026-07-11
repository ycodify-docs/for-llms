# es-n · exemplos anotados

> O es-n não recebe payloads de cliente; o que o agente controla é **como o evento é modelado** (no
> `.model.json`), pois isso determina o despacho. Guia: [README.md](README.md).
> Forma completa do modelo: [examples/](../examples/README.md).

> ⚠️ Blocos abaixo são **ilustrativos**: `<...>` e `•••` são placeholders, não JSON literal pronto para envio.

## Marcadores de despacho no modelo do evento (conceitual)

Todo o roteamento de despacho fica sob **`domainBus`**, dentro da definição do evento (ver a gramática
completa em [model-format](../persistence-crs/spec/model-format.md)):

```jsonc
// evento dentro de "event": { "<nomeDoEvento>": { ... } } no .model.json
"domainBus": {
  "orderingMode": "per-aggregate",          // none | strict | per-aggregate (padrão)
  "triggerProjection": [                      // (opcional) projeção cross-contexto
    { "name": "<alvo>", "targetTenantId": "<tenant destino>", "br": { "route": "<rota>" } }
  ],
  "triggerCoordination": [                    // (opcional) saga
    { "name": "<alvo>", "targetTenantId": "<tenant destino>", "br": { "route": "<rota>" } }
  ]
}
```

- **Sem** marcadores extras → despacho **só** para a projeção do próprio contexto (caminho comum).
- Com `triggerProjection` → também atualiza projeção de outro contexto/tenant.
- Com `triggerCoordination` → também dispara uma saga (consumida pelo persistence-crs → br-service).

## Acionar processamento de um tenant (operacional)

```
POST /api/event-processor/tenants/<tenant-id>/process
→ 200 { "status": "success", "message": "...", "tenantId": "<tenant-id>" }
```

## Efeito observável
Após o despacho, a **projeção** no banco de leitura reflete o novo estado e pode ser consultada via
[persistence-q](../persistence-q/README.md). Eventos com `triggerCoordination` podem gerar **novos
comandos** automaticamente (saga), visíveis como novos eventos no histórico do agregado-alvo.
