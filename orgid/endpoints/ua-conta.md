# orgid · `/ua` · conta

> Domínio **externo** (`/ua`) — contas dos usuários finais dos sistemas. **Senha nunca é devolvida.**
> Registro e recuperação são públicos → [publico.md](publico.md). Guia: [../README.md](../README.md).
>
> ⚠️ **A conta nasce `PENDING` e já é usável — o sign-in NÃO checa o `status`.** A ativação pelo fluxo de
> hash serve para provar posse do e-mail, **não** para liberar o login: uma conta que nunca foi ativada
> autentica normalmente. Se o seu sistema exige conta ativada, o portão é seu, no seu BFF — não conte com
> o orgid para barrar.
>
> **Comum:** `Authorization` obrigatório; POST/PUT enviam `Content-Type: application/json`. Erros: [../erros.md](../erros.md).
>
> **Campos da conta `/ua`** (cadastro externo, além de `username`/`name`/`email`/`status`/endereço):
> `foneUm`, `foneDois`, `whatsapp`, `genero`, `sexo`, `nascimento` (data), `cpf`, `pai`, `mae`,
> endereço pessoal (`endpRua`,`endpNumero`,`endpComplemento`,`endpBairro`,`endpCidade`,`endpUf`,`endpCep`,`endpPais`),
> `habilitacaoNumero`, `habilitacaoValidade` (data), `rgNumero`, `rgOrgaoEmissor`.

## Contents
- GET /ua/account/by/jwt — conta autenticada
- PUT /ua/account — atualizar perfil
- PUT /ua/account/password — atualizar senha
- GET /ua/account/by/username/{username}/role-owner/{roleOwner}/authority — ler conta (por autoridade)
- GET /ua/accounts/by/role-name/{roleName}/role-owner/{roleOwner} — listar por papel
- GET /ua/accounts/by/role-owner/{roleOwner} — listar por dono de papel

---

## GET /ua/account/by/jwt
Retorna a conta do **portador do token**. Sem parâmetros. **Auth:** token válido (sem papel específico).

**Resposta:** `200` — conta externa (campos acima; `password`/`accountRoles` nulos); `204` se não encontrada.

## PUT /ua/account
Atualiza o perfil da própria conta (`username` forçado ao do token). **Auth:** token válido.

**Corpo** (JSON): qualquer subconjunto dos **campos da conta `/ua`** (acima), exceto `username`/`password`.

**Resposta:** `200` (sem corpo) quando gravou · `404` — `"account not found: nothing was updated."`

## PUT /ua/account/password
Troca a senha (`username` forçado ao do token). **Auth:** token válido.

**Corpo** (JSON):

| Campo | Tipo | Obrig. | Significado |
|---|---|---|---|
| `password` | string | sim | Nova senha. |
| `oldPassword` | string | não | Senha atual (validação). |

**Resposta:** `200` (sem corpo) quando trocou · `400` se a senha nova não tem formato adequado · `403` se
a `oldPassword` não confere · `404` — `"account not found: the password was not changed."`

> ⚠️ Até 2026-09-05 a conta inexistente respondia **`204`**, que é sucesso: o usuário recebia confirmação e
> a senha não mudava. Se você ainda vir `204` aqui, está falando com uma versão antiga do orgid.

## GET /ua/account/by/username/{username}/role-owner/{roleOwner}/authority
Lê uma conta no escopo de um dono de papel. **Papéis:** administrador / engenheiro (no `roleOwner`).

| Path-var | Significado |
|---|---|
| `username` | conta a consultar |
| `roleOwner` | dono do papel (espaço de nomes) que autoriza a consulta |

**Resposta:** `200` — conta (senha nula); `204` se não encontrada.

## GET /ua/accounts/by/role-name/{roleName}/role-owner/{roleOwner}
Lista contas que têm um papel. **Papéis:** administrador / engenheiro (no `roleOwner`).

| Path-var | Significado |
|---|---|
| `roleName` | nome do papel |
| `roleOwner` | dono do papel |

**Resposta:** `200` — array de contas (senha nula).

## GET /ua/accounts/by/role-owner/{roleOwner}
Lista todas as contas sob um dono de papel. **Papéis:** administrador / engenheiro (no `roleOwner`).

| Path-var | Significado |
|---|---|
| `roleOwner` | dono do papel |

**Resposta:** `200` — array de contas (senha nula); array vazio `[]` se nenhuma (não `204`).
