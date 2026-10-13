# BackOffice 4.8 — padrões de código (controller -> BLL -> DAL)
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: como o código do BackOffice é organizado e escrito, para o agente saber onde mexer e qual padrão seguir ao planejar bug/melhoria.

## 1. Fluxo de uma requisição

`Controller (MVC.UI ou BackOffice.Api)` → `new BLLxxx()` → `IDALxxx` resolvido no Windsor → `DALxxx` → EF6 (`SMarkBackOfficeContext`) **ou** Dapper/ADO sobre `ConfigurationParametersApplicationSingleton.GetConnectString()`.

- **Controllers instanciam a BLL com `new`** (não há injeção em construtor de controller): `using (var bll = new BLLEmpresas()) { ... }`. Alguns esquecem o `using`.
- **BLL** (`src/BackOffice/BLL/<Area>/BLLxxx.cs`): herda `BLLBase<T>`, resolve o repositório no construtor com `WindsorFactory.Resolver<IDALxxx>()` e o atribui a `base.Repositorio`. Implementa `ValidarAdd/ValidarUpdate/ValidarDelete` (abstratos; muitos são vazios). `BLLBase<T>` já traz `Add/Update/Delete/Retrieve/RetrieveSimple` e versões async (`AddAsync`, `RetrieveAsync` com paginação, `CountAsync`, `AnyAsync`).
- **DAL** (`src/BackOffice/DAL/DAL/<Area>/DALxxx.cs`): `class DALxxx : BackOfficeDALBase<T>, IDALBase<T>, IDALxxx`; construtor recebe `ISMarkBackOfficeContext`. `DALBase<T>` implementa CRUD EF6 genérico. Pesquisas pesadas, listagens DataTables e relatórios usam **Dapper com SQL em string** (`using (var connection = new SqlConnection(GetConnectString()))`), muitas com `WITH (NOLOCK)`. SQL fixo também vive em `DAL/Resources/*.resx` (`Cadastros`, `Ferramentas`, `Login`) e `DAL/SQLs/Ferramentas`.
- **Interfaces** `IDALxxx` ficam em `Domain/Interfaces/Repositorios`. **Entidades** em `Domain/Models`; **DTOs** em `Domain/DTO` (`DataTables`, `Filtros`, `API`, `Queues`) e também em `Global/ServicesDTO` (contrato compartilhado com o CRM).
- **Mapeamento EF**: `Infra/Mapping/<X>Map.cs` (`EntityTypeConfiguration<T>`, `ToTable`, `HasColumnName`), carregado por `OnModelCreating` via `AddFromAssembly`. `DbSet`s declarados em `Infra/SMarkBackOfficeContext.cs` + `ISMarkBackOfficeContext`. `AutomaticMigrationsEnabled = false` (não há migrations EF; o schema muda por scripts SQL — ver `banco-de-dados.md`).

**Para adicionar um acesso a dados novo**: método na interface `IDALxxx` (Domain) → implementação no `DALxxx` → método na `BLLxxx` → chamada no controller. DAL novo (classe) é registrado sozinho: o Windsor registra por convenção todo tipo que implementa `IDALBase<>` no assembly da DAL. Tabela nova mapeada no EF exige entidade + `Map` + `IDbSet` no contexto + script SQL.

## 2. IoC (Windsor) — `SiteExpress.SMark.BackOffice.Ioc/WindsorFactory.cs`

- `WindsorFactory.Iniciar()` é chamado no `Application_Start` (MVC, WebForms, Api) e no `OnStart` do JobHost. Dois modos:
  - **Com `HttpContext.Current`** (web): DAL e `ISMarkBackOfficeContext` com `LifestylePerWebRequest` (`#if NETFRAMEWORK`; `LifestyleScoped` no net10); `IUsuarioLogado` (impl. `LoginDTO`) com ciclo de vida custom **por sessão** (`UsuarioLogadoPerSessionLifeStyle`).
  - **Sem contexto HTTP** (JobHost): tudo `LifestyleScoped`; o chamador abre `WindsorFactory.AbrirEscopo()` por item processado.
- Obter dependência: `WindsorFactory.Resolver<T>()` (service locator estático, usado em controllers, BLL e filtros).
- `SMarkBackOfficeContext(IUsuarioLogado)` usa a connection string `SMarkBackOfficeContext` do Parameter Store (aplicação `APIBO`) e o fallback do `ConfigurationManager`; `CommandTimeout` = appSetting `JobHostCommandTimeout` (default 300 s).
- **Usuário logado**: `IUsuarioLogado` (interface em `Global.ServicesDTO.Interfaces`) = identidade do operador do BackOffice na sessão (`idLogin`, `idEmpresa`, `login`, flags de senha expirada). Preenchido em `LoginController.LogarUsuario` com `CopiaPorValor.CopiarValores`.

## 3. Transações e erros

- `Transacao.ObterEscopo()` (`BLL/Transaction.cs`) devolve `TransactionScope` (Required; timeout padrão 120 s, configurável por appSetting). `BLLBase.Add/Update(List)` usam. Fluxos grandes (ex.: `BLLEmpresas.Add`) misturam EF, Route53 e criação de banco **sem** transação distribuída.
- Exceções de negócio: `BusinessException(msg, MensagemTipo)` (`Global.Common.Exceptions`); o controller captura e devolve a mensagem. Mensagens vêm de `Global/General/Resources/ResourceMessage.resx` (`ResourceHelper.GetResourceMessage("CHAVE")`) ou texto literal em português no código. `LoginException` na Api.
- Log: `LogHelper.LogMessage(msg, CategoriaLog.X)` (Global.Common) nos fluxos de login/empresa; Elmah presente na solução (`src/Elmah`). Contexto EF de log: `SMarkLog.edmx`/`SMarkLogContexto` (Domain/Infra). (Destino real dos logs em produção: não verificado.)

## 4. Retorno ao front

- Listas: `Json(DTResult<T>, JsonRequestBehavior.AllowGet)` (DataTables server-side; entrada `DTParameters`).
- Escrita via modal: `Json(new { sucesso = true/false, message })`. Outros controllers usam `success`/`mensagem`/`error`; **confirme as chaves lendo o JS da view** antes de alterar um retorno.
- Formulários clássicos: `ViewModel` (`Areas/*/Models/*VM.cs`) + `ModelState.AddModelError` e retorno da parcial `_Form*`. Validação de senha: `ValidadorSenhaSeguranca.ValidarSenha`. Anti-forgery só aparece em `ConfiguracoesController` (nas demais telas não há `[ValidateAntiForgeryToken]`).
- Conversão entidade↔DTO: `CopiaPorValor.CopiarValores<TDest,TOrig>` (Global.Common.ObjectCopy) e `InjectFrom` (ValueInjecter, usado na Api). Sem AutoMapper.

## 5. Filtros e segurança no MVC (resumo; detalhes em `seguranca-e-acesso.md`)

- Filtros globais registrados: `HandleErrorAttribute` e `VerificarSenhaExpiradaAttribute` (este em `Application_Start`). O `CustomAuthorize` existe em `MVC/Filters` mas está **comentado** em `FilterConfig` e nos controllers.
- Autorização por menu é feita (quando feita) **dentro do `Index` de 11 controllers** da área Ferramentas ("Valida acesso": o path da requisição precisa estar em algum menu dos grupos do usuário). Actions de `List`/POST/ajax não repetem a checagem.

## 6. Integrações chamadas pela BLL/controllers

| Serviço | Classe | Usado para |
|---|---|---|
| AWS Parameter Store | `ServicesProxy/AWS/SSMService` | Configuração por ambiente na subida |
| AWS Route 53 | `Route53Service` | Cria/remove CNAME do subdomínio da empresa (`BLLEmpresas.Add`, trial) — em `#if !DEBUG` no fluxo de trial |
| AWS S3 | `S3BucketService` | Upload de imagem do editor de avisos (`AvisosController.UploadImagemTiny`) |
| MyBI (BI externo) | `ServicesProxy/API/MyBIService` | Criar/editar/excluir usuário e permissões do Dashboard Smark BI (`Empresas` controller) |
| API de filas/e-mail (core) | `APIsCore/QueuesAPI` + `AuthenticationAPI` | Enfileirar e-mail de aviso (`AvisosController.Publicar`) |
| Zimbra | appSettings `Zimbra*` | Criação de e-mail corporativo de usuários (uso exato: não verificado) |
| E-mail SMTP | `Global/General/SendEmail`, `MVC/Services/EmailService.cs` | Licenciamento e avisos (`MVC/Services/EmailService.cs` não tem chamadores no MVC) |

## 7. Convenções úteis

- Nomes em português; prefixos `BLL`, `DAL`, `IDAL`, `VM`, `DTO`. Pastas por área funcional espelham-se em BLL/DAL/Domain (`Login`, `Permissoes`, `Ferramentas`, `Queue`, `TokensIa`...).
- Multi-target: ao alterar BLL/DAL/Domain/Infra/Ioc/ServicesProxy, manter `#if NETFRAMEWORK` nos usings de SqlClient e não usar API só do net48 sem guarda (o mesmo código compila para `net10.0`). Build do alvo net10 de BLL: (não verificado se está na CI).
- Tarefas Windows/Jobs: `BLLQueueMessage` (`ReclamarProximos`, `FinalizarSucesso/Falha`, `RecuperarLocksOrfaos`) é o contrato com o JobHost — ver `backoffice-api-e-servicos.md`.
- Onde achar X: login → `BLL/Login/BLLLogin.cs` + `DAL/DAL/Login/DALLogin.cs`; empresas/trial → `BLL/Empresas/BLLEmpresas.cs`; menus/grupos/permissões → `BLL/Permissoes/*` + `DAL/DAL/Permissoes/*`; filas → `BLL/Queue/*`; políticas de senha → `BLL/Seguranca/*`; licenças → `BLL/Ferramentas/BLLLicencas.cs`, `BLLSolicitacaoLicencas.cs`, `Permissoes/BLLLicenciamento.cs`.
