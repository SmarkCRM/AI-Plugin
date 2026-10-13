> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Mapa do projeto SMark CRM

Objetivo: mostrar quais projetos existem, como se referenciam, o que é legado em produção x migração, portas e como subir localmente.

## Regra de escopo
- Alvo padrão = projetos .NET Framework 4.8 com namespace `SiteExpress.SMark.*`. São o que está em produção.
- `Smark.*` (net9.0/net10.0) é a migração em andamento (strangler). Só mexer com pedido explícito.
- `src/BackOffice` (MVC 4.8 e WebForms) tem agente próprio; aqui só entram os pontos onde o CRM depende dele.
- Raiz do repo: `CLAUDE.md`, `azure-pipelines.yml`, `containers/`, `infra/`, `doc/`, `src/`.

## Projetos em uso (todos em `src/`)
| Projeto | Caminho | TFM | Papel |
|---|---|---|---|
| SiteExpress.SMark.UI | `SMark/Web/SiteExpress.SMark.UI.csproj` | net48, csproj antigo (packages.config) | Web do CRM (ASP.NET MVC 5.2.8, Areas, SignalR 2.4.3, Hangfire opcional). Porta 62662 (IISUrl no csproj) |
| SiteExpress.SMark.BLL | `SMark/SiteExpress.SMark.BLL` | net48;net10.0 (SDK) | Regras de negócio (508 .cs). Interfaces em `BLL/Interfaces` |
| SiteExpress.SMark.DAL | `SMark/SiteExpress.SMark.DAL` | net48;net10.0 | Acesso a dados: EF6 + Dapper + ADO (226 .cs) |
| SiteExpress.SMark.Domain | `SMark/SiteExpress.SMark.Domain` | net48;net10.0 | Entidades (`Models`, ~200), enums (`Enumeradores/Enumeradores.cs`, 2.621 linhas), `DTO`, `Interfaces/Repositorios` |
| SiteExpress.SMark.Infra | `SMark/SiteExpress.SMark.Infra` | net48;net10.0 | `SMarkContext` (EF6), 175 `Mappings/*Map.cs`, `Conventions/DateConvention` |
| SiteExpress.SMark.Ioc | `SMark/SiteExpress.SMark.Ioc` | net48;net10.0 | `WindsorFactory` (Castle Windsor 5.1.1) e lifestyles customizados |
| SiteExpress.SMark.ServicesProxy | `SMark/ServicesBackOffice` | net48;net10.0 | Clientes HTTP: BackOffice.Api (`Api/`) e integrações externas. Referencia só a DAL |
| SiteExpress.SMark.Api.DTO | `SMark/SiteExpress.SMark.Api.DTO` | net48;net10.0 | DTOs de API |
| SiteExpress.Global.Common | `Global/General` | net48;net10.0 | Enums globais, exceções, resources `Resource*.resx`, Parameter Store, `CopiaPorValor` |
| SiteExpress.Global.ServicesDTO | `Global/ServicesDTO/SiteExpress.Global.ServicesDTO` | net48;net10.0 | DTOs compartilhados (sessão, permissões, `*SearchVMDTO`, fila) |
| SiteExpress.SMark.BackOffice.Api | `Services/SiteExpress.SMark.BackOffice.Api` | net48 (Web API 2) | API que o CRM consome (login, empresas, grupos, licenças, fila). Porta 1999 (IISUrl) |
| SiteExpress.SMark.JobHost | `SiteExpress.SMark.JobHost` | net48;net9.0-windows | Serviço Windows `JobHostServices` |
| SiteExpress.SMark.JobHost.Process | `SiteExpress.SMark.Services` (pasta com nome diferente do projeto) | net48;net9.0-windows | Processos do JobHost (fila, trial, BI) |
| SiteExpress.SMark.Api | `SMark/SiteExpress.SMark.Api` | net48 | API REST legada (v1/v2: Mobile, Trial, IntegracaoERP, IntegracaoSite, Renner, FFM...). 43 controllers. Na solution |
| SiteExpress.SMark.Tests | `Tests/SiteExpress.SMark.Tests` | net48;net10.0 | Único projeto de teste na solution (17 arquivos .cs; xunit + MSTest + Playwright) |

Referências (conferidas nos csproj):
- UI -> BLL, DAL, Domain, Infra, Ioc, ServicesProxy, Global.Common, Global.ServicesDTO.
- BLL -> DAL, Ioc, ServicesProxy, Api.DTO, Global.*. Mais `BackOffice/UI/SiteExpress.SMark.BackOffice.UI.csproj` só no net48: a BLL usa o namespace `SiteExpress.SMark.BackOffice.UI.Util` (ex.: `BLLBeneficiarios`, `BLLValidacoes`, `BLLGerenciarTarefas`; há um `Compatibility/UIHelper.cs` para o net10).
- DAL -> Domain, Infra, Api.DTO, Global.*. Domain -> Api.DTO, Global.*. Infra -> Domain, Global.*. Ioc -> DAL, Domain, Infra, Global.ServicesDTO.
- BackOffice.Api -> BackOffice.BLL/Domain/Infra/Ioc/ServicesProxy **e também** SMark.BLL e SMark.ServicesProxy.
- JobHost.Process -> BackOffice.BLL/Domain/Ioc **e** SMark.BLL/DAL/Domain/Ioc/ServicesProxy. O JobHost usa o `WindsorFactory` do `BackOffice.Ioc` (tem `AbrirEscopo`), não o do CRM.
- `Smark.UI` (net10) referencia as mesmas BLL/DAL/Domain/Infra/Ioc/ServicesProxy. Por isso elas são multi-target.

Consequência: todo código novo em BLL/DAL/Domain/Infra/Ioc/ServicesProxy/Global.* compila também em net10.0. `System.Web`, `HttpContext.Current`, `System.Data.SqlClient` exigem `#if NETFRAMEWORK` (padrão já usado em `WindsorFactory` e DAL).

## Migração (fora do alvo padrão; detalhe curto em `migracao.md`)
- `SMark/Smark.UI` (net10.0): só login/SSO, home e `DeSistema` (4 controllers) contra 152 do legado; migração do CRM interrompida. `Yarp.ReverseProxy` está no csproj mas `Program.cs` não o usa.
- `SMark/Smark.Api` (net9.0), `Services/Smark.BackOffice.Api` (net9.0), `BackOffice/Smark.BackOffice.UI` e `Smark.BackOffice.MVC.UI` (net9.0). Todos na solution.
- Docker (`containers/smark-ui.Dockerfile`, `infra/smark-ui/docker`, `doc/DOCKER-SMARK-UI.md`) é só do Smark.UI.

## Fora da solution (confirmado por grep na .sln)
- `SMark/SiteExpress.SMark.Ioc.Unity`: não referenciado em lugar nenhum. Não usar.
- `SMark/SiteExpress.SMark.Api.Core` e `SiteExpress.SMark.UI.Core`: pastas só com `bin/obj`, sem fonte.
- `SMark/SiteExpress.SMark.UI.Tests` e `SiteExpress.SMark.UI.Functional.Tests`: net4.5, fora da solution (tratar como mortos).
- `SMark.DBDeployer` (WinForms 4.5), `Consoles/*`, `Consoles.Importacao/*`, `Migracao/*`, `ProjetosAuxiliares/*`, `Services/BackOfficeServices*`, `Elmah`: utilitários antigos. O que cada um faz não foi verificado.

## Solution e build
- `src/Smark_SolutionCompleta.sln`: 29 projetos. Configurações: `Debug`, `Release`, `aws_dev`, `aws_hlg` (Any CPU, x86, x64).
- Pacotes `packages.config` vão para `src/packages` (não versionada). MSBuild do Visual Studio, sem `/restore`, nos projetos antigos (UI, BackOffice.Api, MVC). SDK-style (BLL/DAL/etc.) restauram normalmente.
- `MvcBuildViews=false`: o build não compila `.cshtml`. Erro de Razor só aparece ao abrir a tela.
- UI é csproj antigo: todo `.cs`/`.cshtml`/`.js` novo precisa de `<Compile Include>`/`<Content Include>` (464 Compile e 665 Content de views no csproj hoje). BLL/DAL/Domain/Infra incluem arquivos automaticamente.
- CI: `azure-pipelines.yml` (`trigger: none`, `windows-latest`, Release): NuGet restore -> VSBuild (publica `WebApp.zip`) -> VSTest. Hook de segredos em `src/.githooks/pre-commit` (ativar com `git config core.hooksPath .githooks`).
- Branch/commit: `feat/...`, `fix/...`, `patch/x.y.z`; commits `feat(#NNNNN): descrição`. Os guias `doc/conventions/CHANGESET.txt` e `doc/guides/MERGE.txt` são da era TFS (históricos).

## Como rodar localmente (CLAUDE.md + csproj)
Três processos juntos:
1. Redis: `W:\Smark\redis-latest\redis-server.exe`, porta 6379 (`Redis:Server`/`Redis:Port` no `Web.config`). Não instalar outro.
2. `SiteExpress.SMark.BackOffice.Api` em http://localhost:1999/ (`BackOfficeServicesApiBaseUrl` aponta para ele).
3. `SiteExpress.SMark.UI` em http://localhost:62662/.
Opcionais: JobHost (em `DEBUG` roda como console, thread `JobHostProcess`) e BackOffice MVC `SiteExpress.Smark.BackOffice.MVC.UI` em http://localhost:49981/.
Configuração vem do AWS Parameter Store (chave `AWSParameterStoreEnvironment`, um ambiente por dev: `env-local<Nome>`). Os configs versionados hoje trazem valores de dev diferentes (UI `env-localPedro`, JobHost `env-localCassio`, Api legada `env-localRegis`). Não commitar a troca.
`src/.vs/.../applicationhost.config` (portas IIS Express) é ignorado pelo git.

## Onde estão as coisas (atalhos)
- Tela -> `SMark/Web/Areas/<Area>/{Controllers,Views,Scripts}`; ver `areas-e-modulos.md`.
- Regra -> `SiteExpress.SMark.BLL/<Modulo>/BLL<X>.cs`; SQL/consulta -> `SiteExpress.SMark.DAL/<Modulo>/DAL<X>.cs`.
- Tabela do tenant -> `Domain/Models/<X>.cs` + `Infra/Mappings/<X>Map.cs`.
- Script de banco -> `src/BackOffice/UI/Sql/Scripts por Sprint/<versão>/`.
- Texto de tela -> `SMark/Web/App_GlobalResources/{Label,Mensagens,Enumeradores}.resx`.
- Fila/job -> `SiteExpress.SMark.Services/Process/QueueProcess`.
- Integração externa -> `SMark/ServicesBackOffice/<Integracao>` e `Web/Areas/Integracao`.
