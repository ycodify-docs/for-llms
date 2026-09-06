# gateway — exemplos

> Exemplos do que a **borda** faz com uma requisição antes de ela chegar ao serviço. Para os
> endpoints de cada serviço, ver o `exemplos.md` do serviço.
> Voltar ao [guia da borda](README.md) · [catálogo de erros](erros.md).

## Requisição bem formada

Exatamente **um** cabeçalho de identificação. Rota de execução → `X-Tenant-Id`:

```bash
curl -X POST "$BORDA/v3/persistence/c/e" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: 550e8400-e29b-41d4-a716-446655440000" \
  -d '{ }'
```

Rota administrativa → `X-Forger-Credential`:

```bash
curl "$BORDA/v3/forger/..." \
  -H "X-Forger-Credential: <credencial>"
```

## Os dois cabeçalhos juntos — recusado

```bash
curl "$BORDA/v3/persistence/c/e" \
  -H "X-Tenant-Id: <tenant>" \
  -H "X-Forger-Credential: <credencial>"      # ← os dois: erro
```

Devolve **`400`**, com o motivo no corpo e `X-Blocked-By` nos cabeçalhos. Mandar os dois **não** é
mais permissivo que mandar um: é recusa.

## Distinguir o `404` da borda do `404` do serviço

Inspecione os cabeçalhos de resposta — é a única forma confiável:

```bash
curl -sS -o /dev/null -D - "$BORDA/v3/<servico>/<caminho>" \
  -H "X-Tenant-Id: <tenant>" | grep -i "^HTTP/\|^x-blocked"
```

| Saída | Leitura |
|---|---|
| `HTTP/1.1 404` **+** `x-blocked-by:` … | a **borda** recusou; o serviço nunca foi chamado |
| `HTTP/1.1 404` **sem** `x-blocked-*` | o **serviço** respondeu; o recurso é que não existe |

## Segmento obrigatório faltando

Vários serviços exigem um segmento além do nome. O caminho abaixo parece plausível e mesmo assim
morre na borda:

```bash
# sem o segmento obrigatório depois do serviço → 404 DA BORDA
curl "$BORDA/v3/persistence/logs/..."   -H "X-Tenant-Id: <tenant>"

# com o segmento → chega ao serviço
curl "$BORDA/v3/persistence/c/logs/..." -H "X-Tenant-Id: <tenant>"
```

Se o `404` trouxer `X-Blocked-By`, é este caso — não adianta investigar o serviço.

## Corpo grande

```bash
curl -X POST "$BORDA/v3/<servico>/<caminho>" \
  -H "X-Forger-Credential: <credencial>" \
  --data-binary @arquivo.zip
```

`413` com `X-Blocked-By` veio da borda; `413` sem ele veio do proxy à frente dela. O teto **varia por
serviço**: rotas que recebem carga binária têm teto maior que rotas de JSON.

## Verificar se a borda está no ar

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$BORDA/"
```

Responder **não** significa que a plataforma está sã: a borda pode estar de pé com serviços
indisponíveis atrás dela — nesse caso as rotas devolvem `503`. Para saber se um serviço específico
responde, chame um endpoint dele.
