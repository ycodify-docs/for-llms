# orgid · manifesto da aplicação (`/up/application`, `/open/application`, `/ua/application`)

> A **identidade da aplicação** de uma organização: nome, subtítulo, ícone, esquema de cor, autocadastro,
> termos de acesso e política de privacidade. A casca a acha pelo **domínio** por onde o navegador chegou,
> antes do login. Guia: [../README.md](../README.md). Erros: [../erros.md](../erros.md).
>
> **Em produção desde 2026-10-16T14:34:24Z.** Pela borda, as rotas de leitura chegam ao orgid: `GET
> /open/application` responde `404` a domínio não verificado, e `GET /ua/application/{org}` sem token responde
> `401`. Ainda não foi medido em produção publicar um manifesto e provar um domínio pelo TXT.

## Contents
- Quem publica e quem lê
- Como o domínio casa
- POST /up/application/{org} — publicar o manifesto
- GET /up/application/{org} — estado completo, para quem publica
- POST /up/application/{org}/domain — registrar domínio
- POST /up/application/{org}/domain/{dominio}/verify — provar a posse (TXT)
- DELETE /up/application/{org}/domain/{dominio} — remover domínio
- POST /up/application/{org}/text/{termos|privacidade} — publicar versão de texto
- GET /open/application?domain={host} — ler pelo domínio (público)
- GET /ua/application/{org} — ler pela org (com sessão)
- Pitfalls

---

## Quem publica e quem lê

| Rota | Credencial |
|---|---|
| `/up/application/{org}/**` | token de **plataforma** com **administrador** (`API_MASTER`) em `{org}`. Engenheiro não publica → `403` |
| `GET /open/application?domain=` | **nenhum token**: é a leitura antes do login. Na borda vale o cabeçalho de identificação de sempre |
| `GET /ua/application/{org}` | token de **aplicação** de um usuário que tem papel em `{org}`; fora dela → `403` |

`{org}` é o nome da organização (o mesmo de `/org/{org}` no forger), em minúsculas.

## Como o domínio casa

- **`*.exemplo.com.br`** casa os **subdomínios** (`app.exemplo.com.br`, `a.b.exemplo.com.br`) e **não** casa
  `exemplo.com.br`.
- **Sem `*`**, casa **só** o host exato.
- Vence o **mais específico**: o host exato, depois o curinga de sufixo mais longo.
- O mesmo domínio **não** pode estar em duas organizações (`409` no registro).
- **Só domínio com posse provada casa.** A prova é um registro **TXT** no DNS:

| Domínio registrado | Nome do TXT | Valor |
|---|---|---|
| `app.exemplo.com.br` | `_yc-verificacao.app.exemplo.com.br` | `yc-verificacao=<token>` |
| `*.exemplo.com.br` | `_yc-verificacao.exemplo.com.br` (no domínio-base) | `yc-verificacao=<token>` |

O `<token>` vem na resposta do registro e não muda ao repetir o registro.

## POST /up/application/{org} — publicar o manifesto

Cria ou substitui o manifesto. Cada publicação sobe a **versão** e fica registrada (quem, versão, quando).

**Corpo** (JSON) — chave fora desta lista é **recusada** (`400`), não ignorada:

| Campo | Tipo | Obrig. | Regra |
|---|---|---|---|
| `nome` | string | sim | até 120 caracteres |
| `subtitulo` | string | não | até 240 caracteres |
| `icone` | string | não | data URI `data:image/svg+xml`, `data:image/png;base64` ou `data:image/webp;base64`; até **32 KiB** (`413`). SVG com script, atributo de evento (`on…=`), `javascript:` ou referência externa é recusado; referência interna (`url(#id)`, `href="#id"`) é aceita. PNG e WEBP têm a assinatura conferida |
| `esquema` | string | sim | `verde`, `azul`, `ambar`, `roxo` ou `cinza` |
| `autocadastro` | boolean | sim | a tela de login oferece "criar conta" |

**Resposta:** `201 {"versao": <n>}` · `400` forma · `403` sem administrador · `404` org inexistente · `413` ícone grande.

## GET /up/application/{org} — estado completo

Para quem publica conferir. **Resposta** `200`:

```json
{
  "org": "exemplo",
  "manifesto": { "nome": "…", "subtitulo": null, "icone": null, "esquema": "verde", "autocadastro": false, "versao": 2 },
  "dominios": [
    { "dominio": "*.exemplo.com.br", "verificado": false, "verificadoEm": null,
      "txt": { "nome": "_yc-verificacao.exemplo.com.br", "valor": "yc-verificacao=<token>" } }
  ],
  "termos": [ { "versao": 1, "vigenteDesde": 1760600000000 } ],
  "privacidade": []
}
```

`manifesto` é `null` enquanto nada foi publicado.

## POST /up/application/{org}/domain — registrar domínio

**Corpo:** `{"dominio": "app.exemplo.com.br"}` ou `{"dominio": "*.exemplo.com.br"}`. Maiúsculas e ponto final
são normalizados. Recusados (`400`): porta, caminho, endereço IP, nome de um rótulo só, `*` fora do começo,
curinga com um rótulo só depois do `*.`.

**Resposta:** `201 {"dominio", "verificado": false, "txt": {"nome", "valor"}}` — crie o TXT e chame o verify.
Repetir o mesmo domínio na mesma org devolve o mesmo token · `409` se outra organização já tem o domínio.

## POST /up/application/{org}/domain/{dominio}/verify — provar a posse

O orgid consulta o TXT agora. **Resposta** sempre `200`:

| Corpo | Quando | Efeito |
|---|---|---|
| `{"verificado": true, "verificadoEm": <epoch ms>}` | o TXT tem o valor | o domínio passa a casar |
| `{"verificado": false, "motivo": "…"}` | o TXT respondeu sem o valor | **a posse deixa de valer**, mesmo que valesse antes |
| `{"verificado": false, "motivo": "DNS lookup … failed …"}` | a consulta falhou | nada muda |

`404` se o domínio não está registrado nesta org. A propagação de DNS pode atrasar: repita depois.

## DELETE /up/application/{org}/domain/{dominio} — remover domínio

**Resposta:** `200 {"deleted": true}` — diz que **não resta** o domínio, tenha havido ou não o que remover
(o gateway repete DELETE).

## POST /up/application/{org}/text/{termos|privacidade} — publicar versão de texto

Texto **simples** (não HTML). Toda publicação é uma **versão nova**; versão publicada **não muda** e não se
remove — é a ela que o aceite do usuário vai apontar.

**Corpo:** `{"texto": "…", "vigenteDesde": <epoch ms, opcional>}`. Sem `vigenteDesde`, vale a partir de agora;
com data futura, a versão anterior segue vigente até lá. Texto até **100 KiB** (`413`).

**Resposta:** `201 {"tipo", "versao", "vigenteDesde"}` · `400` tipo, chave ou texto vazio.

## GET /open/application?domain={host} — ler pelo domínio

`{host}` é o nome por onde o navegador chegou (`location.hostname`), sem porta. **Público.**

**Resposta** `200`:

```json
{
  "org": "exemplo",
  "nome": "Exemplo",
  "subtitulo": null,
  "icone": "data:image/svg+xml;base64,…",
  "esquema": "verde",
  "autocadastro": true,
  "termos": { "versao": 3, "vigenteDesde": 1760600000000, "texto": "…" },
  "privacidade": null
}
```

`termos`/`privacidade` trazem a **vigente** (a maior versão já em vigor) ou `null`. A lista de domínios
**não** é devolvida. `404` quando nenhum domínio **verificado** casa, a org não existe mais, ou ela ainda não
publicou manifesto · `400` sem `domain` ou com host inválido.

## GET /ua/application/{org} — ler pela org

Mesmo corpo da leitura por domínio, para o usuário já logado (troca de org). `403` se `{org}` não está
entre as orgs do token · `404` sem manifesto.

## Pitfalls

- **Domínio registrado não é domínio que casa.** Sem o verify com o TXT certo, a leitura pública dá `404`.
- **O TXT do curinga fica no domínio-base**, não em `*.…`.
- **Remover o TXT e chamar o verify desfaz a posse**; não chamar o verify não desfaz.
- **`/ua/open/…` é público** no orgid: a leitura por org, que exige token, é `/ua/application/{org}`.
- **Os textos são simples**: exiba com quebra de linha preservada, nunca como HTML.
