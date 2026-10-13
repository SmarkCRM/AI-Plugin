> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Migração (projetos Smark.* net9/net10)

Objetivo: dizer o que existe da migração do CRM, que ela é pequena e interrompida, e qual regra seguir no dia a dia.

## O que existe (todos na `Smark_SolutionCompleta.sln`)
| Projeto | TFM | Conteúdo hoje |
|---|---|---|
| `src/SMark/Smark.UI` | net10.0 | 64 `.cs`. Controllers: `LoginController`/`LoginNovoController` (login que grava o cookie `SMSSO` e o payload SSO no Redis), `Areas/Home` (`HomeController`, `HomeNovoController`, logout), `Areas/Configuracoes/DeSistemaController` (101 linhas). `Services/MenuBuilderService`, `PermissaoService`, middlewares (erro, paginação, DI). Redis (cache + DataProtection), Hangfire, EF Core no csproj. `Yarp.ReverseProxy` está no csproj, mas `Program.cs` não o usa (verificado por grep); roteamento para o legado não verificado |
| `src/SMark/Smark.Api` | net9.0 | `Program.cs` de 29 linhas, praticamente vazio |
| `src/Services/Smark.BackOffice.Api`, `src/BackOffice/Smark.BackOffice.UI`, `Smark.BackOffice.MVC.UI` | net9.0 | Parte do BackOffice; fora do escopo deste agente (BackOffice migrou para outro repositório) |

Comparação: o legado `SiteExpress.SMark.UI` tem 152 controllers em 9 Areas; o `Smark.UI` tem 4 controllers (login/home/DeSistema). Ou seja, a migração do CRM cobre login/SSO e pouco mais; está interrompida (informação do time; o código confirma o tamanho pequeno).

## Como se relacionam
- BLL, DAL, Domain, Infra, Ioc, ServicesProxy, Global.Common e Global.ServicesDTO são **multi-target `net48;net10.0`**; o `Smark.UI` referencia os mesmos projetos. É o único acoplamento real entre os dois mundos.
- SSO: o `Smark.UI` grava `SMSSO` + `{Redis:ApplicationName}sso:{token}`; o legado lê e restaura a sessão (ver `seguranca-sessao-config.md`). Alterar `LoginSessionDTO`/`MenusSessionDTO` afeta os dois.

## Regras práticas
1. Bug ou melhoria do CRM: trabalhar no **legado 4.8** (`SiteExpress.SMark.*`, `src/SMark/Web`), salvo pedido explícito de mexer em `Smark.*`.
2. Alterar BLL/DAL/Domain/Infra/Ioc/ServicesProxy/Global.*: o código precisa compilar também em **net10.0**. `System.Web`, `HttpContext.Current`, `System.Data.SqlClient`, APIs só de .NET Framework exigem `#if NETFRAMEWORK` (padrão em `WindsorFactory` e DAL). Validar com build da solução ou `dotnet build -f net48` e `-f net10.0` do projeto alterado.
3. Mudança de comportamento de regra compartilhada (BLL/DAL) vale para o `Smark.UI` também; avisar no plano.
4. Mudança em login/sessão/SSO: desenhar para legado e `Smark.UI` juntos.
5. Não criar telas novas no `Smark.UI` por "ser o novo" sem pedido: as telas em produção são as do legado.
