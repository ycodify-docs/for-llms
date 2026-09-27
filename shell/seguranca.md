# shell · segurança

> As invariantes de segurança da casca + BFF. Regra-mãe: **todo segredo vive no BFF; o browser nunca vê
> token nem chave**. A casca é a interface; o **BFF** é o guardião. Pré: [README](README.md),
> [bff](bff.md), [autenticação](../06-autenticacao.md).
>
> _(Documento **público**: descreve o **modelo**, não valores. Nenhuma credencial, chave, senha ou
> endereço sensível deve constar aqui — isso é operacional e fica **fora** desta documentação.)_

## Invariantes

- **Zero segredo no browser.** O token de acesso obtido no [auth](../auth/README.md) fica **num store de
  sessão server-side** — o BFF é **stateless** em memória. O **cookie httpOnly** guarda apenas um
  **identificador de sessão opaco** (não o token). As respostas do BFF **nunca** trazem o token no corpo;
  cada requisição resolve a sessão pelo identificador.
- **O BFF injeta as credenciais no servidor.** `Authorization: Bearer …` e `X-Tenant-Id` são compostos
  **pelo BFF** ao falar com a plataforma. A casca e o miolo **não** montam esses cabeçalhos.
- **Capacidade = UX, não segurança.** A casca **esconde** o que o papel não autoriza, mas o
  [persistence-crs](../persistence-crs/README.md) **revalida** cada comando no servidor (defesa em
  profundidade). Esconder na UI **nunca** é a fronteira de segurança.
- **Só o tenant do usuário.** A casca só resolve capacidade/miolo de um tenant que **consta no token**
  do usuário; pedir outro → recusado.
- **Papéis org-scoped.** A autorização é **por organização**: um papel numa org **não** vale em outra
  (ver [autenticação](../06-autenticacao.md)). A capacidade usa os papéis **da org do tenant** selecionado.

## Fronteiras (quem sabe o quê)

| Ator | Vê o token? | Compõe `Authorization`/`X-Tenant-Id`? | Fala com a plataforma? |
|---|---|---|---|
| **browser / casca** | não | não | não — só com o BFF |
| **miolo** | não | não | não — só com o BFF (via `api` do hostContext) |
| **BFF** | sim (cookie httpOnly) | **sim** (no servidor) | sim (via gateway) |

## Sessão

- Login novo **invalida** o anterior (sessão única — ver [auth](../auth/README.md)); token de **curta
  duração**. Ao expirar (`401`), a casca trata como **re-login**.
- Encerrar sessão limpa o cookie no BFF.

## Diagnóstico do relato de suporte

O relato de suporte leva o **diagnóstico da tela**: o que a casca registrou na aba para quem for investigar.
Fica só na memória da aba, e **nada sai sem o usuário relatar** — ele vê cada parte na gaveta de suporte,
desmarca o que não quiser enviar e pode copiá-lo.

| Parte | O que leva | O que **nunca** leva |
|---|---|---|
| **erros e avisos** (até 20 de cada) | origem (casca, miolo, rede, injeção), mensagem, pilha, pilha de componentes, instante, tela; as últimas linhas de erro e aviso do console | token, cookie, senha |
| **chamadas ao servidor** (até 30) | método, caminho com **só as chaves** da query, status, duração, `x-request-id` | corpo da requisição ou da resposta, valor de parâmetro |
| **o que foi feito na tela** (até 50) | org, contexto, agregado, comando aberto ou recusado, **nomes** dos campos filtrados, **id** do registro selecionado | valor de campo, texto digitado |
| **versões e ambiente** | versão da casca e do miolo, navegador, tamanho da tela, tema, esquema, tenant, papel ativo, fuso | — |

- **Todo texto passa por um redator** antes de entrar: JWT, `Bearer …`, e-mail, CPF e sequência longa de
  letras e números saem trocados por marcadores. Mensagem até 500 caracteres, pilha até 4000, diagnóstico
  inteiro até 64 KB — acima disso sai primeiro o mais antigo e o que menos explica.
- **O `x-request-id` liga o relato à linha do log do BFF** ([bff](../bff/README.md#id-de-requisição-x-request-id)).
  Não é credencial: não autentica nada e não carrega dado do usuário.
- **O miolo contribui por `hostContext.diagnostico`** só com nomes e ids ([contrato](contrato-miolo.md)).

## Checklist do agente

- [ ] Nunca expor o token à casca/miolo — **cookie httpOnly**, corpo sem token.
- [ ] Nunca compor `Authorization`/`X-Tenant-Id` no browser — é o **BFF**.
- [ ] Nunca tratar o gating da UI como segurança — o servidor **revalida**.
- [ ] Nunca resolver tenant fora do token do usuário.
- [ ] Nunca escrever segredo/credencial/endereço sensível em documentação pública.
- [ ] Nunca pôr valor de registro, texto digitado ou corpo de requisição no diagnóstico — só nomes e ids.
