# BackOffice interno 4.8 — visão geral
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: dizer o que é o BackOffice interno (.NET Framework 4.8), quem usa, como os projetos se dividem, quem depende de quem, portas e como buildar/rodar. Os demais arquivos desta pasta detalham cada parte.

## 1. O que é

- É o painel **administrativo interno da Smark** (equipe da casa, não o cliente final) que administra o **banco central BACKOFFICE**, compartilhado por todos os clientes (tenants) do CRM. O CRM em si fica em `src/SMark/Web` (`SiteExpress.SMark.UI`, porta 62662 segundo o `CLAUDE.md`).
- Cadastra e controla: empresas/clientes (`Empresas`), trials, licenças (CRM e BI), módulos contratados (`ModulosSistema`), usuários e grupos do próprio BackOffice, usuários dos clientes, permissões por grupo/menu (`Menus`, `Permissoes`), avisos/e-mails para clientes, onboards, controle de tokens de IA, relatórios de engajamento/uso de funcionalidades, auditoria de consulta CNPJ e a fila de processamento do JobHost.
- Modelo **multi-tenant com um banco SQL por empresa**: o BACKOFFICE guarda, por empresa, servidor + nome do banco + usuário do banco (tabela `Empresas`/`Servidores`). O BackOffice abre conexão direta nesses bancos em alguns fluxos (ver `banco-de-dados.md`).
- Tudo aqui é legado .NET Framework 4.8, namespaces `SiteExpress.SMark.BackOffice.*` e `SiteExpress.Smark.BackOffice.MVC.UI`. Existem projetos paralelos net9/net10 (`Smark.BackOffice.MVC.UI`, `Smark.BackOffice.UI`, `Smark.BackOffice.Api`) e um BackOffice novo (.NET 10 + React, repo `smark-backoffice`): **fora do escopo**, só saiba que existem.

## 2. Projetos (src/BackOffice, src/Services)

| Projeto | Caminho | Tipo | Observação |
|---|---|---|---|
| **SiteExpress.Smark.BackOffice.MVC.UI** | `src/BackOffice/SiteExpress.Smark.BackOffice.MVC.UI` | ASP.NET MVC 5, net48, csproj antigo + `packages.config` | Web do BackOffice atual. Porta de dev 49981 (`IISUrl` no csproj). Só está na `Smark_SolutionCompleta.sln`. |
| SiteExpress.SMark.BackOffice.UI (WebForms) | `src/BackOffice/UI` | Web Forms + `.ashx`, net48, porta de dev 58323 | **Legado**, ainda hospeda: runner de migrations (`Views/Migrations/RunMigrations.aspx`), pasta `Sql/` inteira, telas de Menus/Servidores/Conteúdos/Suporte/Log. Está na `BackOffice.sln`. Se ainda é publicado: (não verificado). |
| BLL | `src/BackOffice/BLL` | SDK-style, `net48;net10.0` | Regras de negócio (`BLLxxx`). Compartilhado com BackOffice.Api, JobHost e projetos net10. |
| DAL | `src/BackOffice/DAL` | SDK-style, `net48;net10.0` | Repositórios EF6 + Dapper/ADO; SQL em `SQLs/` e `Resources/*.resx`. |
| Domain | `src/BackOffice/Domain` | SDK-style, `net48;net10.0` | Entidades, DTOs, interfaces `IDALxxx`, enums, `SMarkLog.edmx`. |
| Infra | `src/BackOffice/Infra` | SDK-style, `net48;net10.0` | `SMarkBackOfficeContext` (DbContext EF6) + `Mapping/*Map.cs`. |
| Ioc | `src/BackOffice/SiteExpress.SMark.BackOffice.Ioc` | `net48;net10.0` | `WindsorFactory` (Castle Windsor) e `IUsuarioLogado` por sessão. |
| ServicesProxy | `src/BackOffice/SiteExpress.SMark.BackOffice.ServicesProxy` | `net48;net10.0` | Clientes AWS (SSM, S3, Route53), `MyBIService`, `QueuesAPI`, `AuthenticationAPI`. |
| UI.Core, MVC.UI.Core | `src/BackOffice/...UI.Core`, `...MVC.UI.Core` | só `bin/obj` | Sem código-fonte versionado. |
| **SiteExpress.SMark.BackOffice.Api** | `src/Services/SiteExpress.SMark.BackOffice.Api` | Web API 2, net48, IISUrl `http://localhost:1999/` | API consumida pelo CRM (ver `backoffice-api-e-servicos.md`). |
| BackOfficeServices | `src/Services/BackOfficeServices` | WCF netTcp em Windows Service, net4.5 | Legado; fora das duas solutions principais. |
| BackOfficeServicesControl | `src/Services/SiteExpress.SMark.BackOfficeServicesControl` | WinForms net4.5 | Botões iniciar/parar/reiniciar o serviço acima. |
| Api.Core | `src/Services/SiteExpress.SMark.BackOffice.Api.Core` | só `bin/obj` | Sem código versionado. |
| ServicesProxy do CRM | `src/SMark/ServicesBackOffice` (`SiteExpress.SMark.ServicesProxy`) | net48 | Lado **cliente** da BackOffice.Api, usado pelo CRM, JobHost e BackOffice. |

Não confunda: há dois `ServicesProxy` (CRM em `src/SMark/ServicesBackOffice`, BackOffice em `src/BackOffice/SiteExpress.SMark.BackOffice.ServicesProxy`) e dois `SSMService.cs` (um em cada) — mudança em um deve ser avaliada no outro.

`src/Consoles`, `src/Consoles.Importacao` e `src/ProjetosAuxiliares` têm consoles utilitários pontuais (endereços, upload S3, ajuste de logins) que referenciam o BackOffice; não são parte do fluxo normal.

## 3. Dependências (sentido da seta = "referencia")

```
MVC.UI  -> BLL, Domain, Ioc, ServicesProxy(BO), Global.Common, Global.ServicesDTO,
           e também SiteExpress.SMark.BLL / SMark.Domain (do CRM)
BackOffice.Api -> BLL, Domain, Infra, Ioc, ServicesProxy(BO), ServicesProxy(CRM), SMark.BLL
BLL -> Ioc, ServicesProxy(BO), ServicesProxy(CRM)
Ioc -> DAL -> Infra -> Domain -> Global.ServicesDTO
JobHost / JobHost.Process (src/SiteExpress.SMark.JobHost*) -> BackOffice BLL/Ioc (acesso direto ao BACKOFFICE)
```

Consequência: **BLL/DAL/Domain/Infra/Ioc/ServicesProxy(BO) são compartilhados** pelo BackOffice MVC, BackOffice.Api, JobHost e (via alvo net10.0) pelos projetos de migração. Alterar um método de BLL/DAL pode quebrar outros hosts. Código dependente de plataforma usa `#if NETFRAMEWORK` (usings de `System.Data.SqlClient` vs `Microsoft.Data.SqlClient`, ciclo de vida do Windsor).

## 4. Portas e ambientes

| O quê | Porta/URL local |
|---|---|
| BackOffice MVC | http://localhost:49981/ |
| BackOffice.Api | http://localhost:1999/ (o CRM usa a chave `BackOfficeServicesApiBaseUrl` + `api/v1/`) |
| BackOffice Web Forms | porta 58323 (csproj) |
| CRM (UI) | 62662 (`CLAUDE.md`) |

Configs por ambiente: `Web.Debug/Release.config` no MVC; BackOffice.Api e WebForms têm também `Web.aws_dev.config` / `Web.aws_hlg.config` (transforms XDT). JobHost usa `App.aws_dev/hlg.config`. As configurações reais (conexões de banco, chaves) vêm do **AWS Parameter Store** — ver `seguranca-e-acesso.md`.

## 5. Como buildar e rodar (verificado nos csproj/CLAUDE.md)

- Solution completa: `src/Smark_SolutionCompleta.sln`; a CI (`azure-pipelines.yml`, `trigger: none`) faz restore + `VSBuild` + `VSTest` dessa solution. Há `src/BackOffice/BackOffice.sln` (só BLL/DAL/Domain/Infra/Ioc/ServicesProxy + WebForms) e `src/Services/BackOfficeServices.sln` (BLL..Api, sem o projeto BackOfficeServices em si).
- O MVC.UI e a Api usam `packages.config` (restore via NuGet/Visual Studio); BLL/DAL/Domain/Infra/Ioc/ServicesProxy são SDK-style com `PackageReference` e multi-target — buildar o `net48` para o BackOffice. Comando exato de CLI: (não verificado).
- Rodar o BackOffice MVC localmente: abrir `SiteExpress.Smark.BackOffice.MVC.UI.csproj` no Visual Studio (IIS Express, 49981). Segundo o `CLAUDE.md` é independente do Redis e do CRM. Precisa do Parameter Store acessível: o `Application_Start` chama `SSMService` **sempre** (não consulta `UsarParameterStore`) e a chave `AWSParameterStoreEnvironment` do `Web.config` seleciona o ambiente do desenvolvedor — não commite essa troca.
- O MVC **não tem connection string ativa no Web.config** (estão comentadas); a conexão do contexto `SMarkBackOfficeContext` vem do Parameter Store (aplicação `APIBO`). Sem AWS = sem banco.
- Front: `package.json` declara `gentelella ^1.3.0`, mas `node_modules` está **versionado** no repo (gentelella, tinymce...). Não precisa rodar `npm install` para subir. `bundleconfig.json` (Bundler & Minifier) só gera `.min` de arquivos do gentelella/Content; o `BundleConfig.cs` do MVC está todo comentado (as views referenciam scripts direto).
- Para acionar o JobHost pelo BackOffice (`ListaJobhosts`), o IIS do BackOffice precisa estar na mesma máquina do serviço Windows (usa `ServiceController` local).

## 6. Onde olhar primeiro

| Dúvida | Arquivo |
|---|---|
| Rotas/telas | `modulos-e-telas.md` |
| Como o código é escrito | `backend.md` |
| Tabelas e scripts SQL | `banco-de-dados.md` |
| API, proxy, serviços, filas | `backoffice-api-e-servicos.md` |
| Login, perfis, segredos | `seguranca-e-acesso.md` |
| Problemas conhecidos | `armadilhas-e-divida-tecnica.md` |
| Checklists | `roteiros.md` |
