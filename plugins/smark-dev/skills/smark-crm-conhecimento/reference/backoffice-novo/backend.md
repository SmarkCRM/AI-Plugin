# BackOffice novo — backend

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: mapa da solução .NET (camadas, DI, minimal APIs, EF Core sobre tabelas legadas, auditoria, integrações) para localizar onde mexer num bug ou melhoria.

Raiz: `backend/` — solução `SmarkBackOffice.slnx`, `Directory.Build.props` (`TreatWarningsAsErrors=true`, Nullable, ImplicitUsings), `.editorconfig`, `dotnet-tools.json` (dotnet-ef 10.0.11, mas **não há migrations**).

## Camadas (dependências apontam para dentro)

| Projeto | Papel | Pacotes notáveis |
|---|---|---|
| `SmarkBackOffice.Domain` | entidades e regras puras (`Entity` com `Id` int), enums (auditoria, fila, trials) | nenhum |
| `SmarkBackOffice.Application` | casos de uso/serviços, DTOs, interfaces (`I*Repository`, gateways), validadores FluentValidation, `*Options` | FluentValidation 12, Options |
| `SmarkBackOffice.Infrastructure` | `AppDbContext`, configurations EF, repositórios, tenant (SQL direto), AWS, e-mail, HTTP, workers | EF Core SqlServer 10.0.11, AWSSDK (S3, Route53, SSM), HtmlSanitizer, ServiceController |
| `SmarkBackOffice.Api` | borda HTTP: `Program.cs`, `Endpoints/`, `Autorizacao/`, `Auditoria/`, `Scripts/` (SQL) | OpenApi, Scalar |

Pastas por módulo espelhadas em Domain/Application (ex.: `Avisos/`, `Trials/`, `FilaJobHosts/`). Infrastructure agrupa por tipo: `Persistence/Configurations`, `Persistence/Repositories`, `Tenant/`, `Aws/`, `CrmApi/`, `Filas/`, `Integracoes/MyBi/`, `JobHost/`, `Email/`, `Dashboard/`, `Auditoria/`, `Autorizacao/`, `Configuracao/`.

## Program.cs (ordem relevante)

`AddOpenApi` → `AddApplication` → `AddInfrastructure(configuration)` → `IContextoAuditoria` HTTP (`ContextoAuditoriaHttp`) → ForwardedHeaders (só confia em `Auditoria:ProxiesConfiaveis`; vazio = headers desligados) → cookie auth → **todas as policies inline** (um bloco grande no `Program.cs`) → CORS (`Cors:Origins`, `AllowCredentials`) → `UseForwardedHeaders`, `UseDefaultFiles`, `UseStaticFiles` → OpenAPI/Scalar **só em Development** → `UseCors`, `CorrelationIdMiddleware`, `UseAuthentication/Authorization` → `/health` → `Map<Modulo>Endpoints()` ×14 → `MapFallbackToFile("index.html")`. A classe `public partial class Program` existe para os testes (`WebApplicationFactory<Program>`).

Não há `UseExceptionHandler`/ProblemDetails global: cada endpoint trata erros (`ValidationProblem`, `ProblemHttpResult`, `ErroDto`); `LogsAuditoriaEndpoints` tem um endpoint filter próprio que devolve 500 genérico sem `ex.Message`.

## Padrão de endpoint

Um arquivo `Endpoints/<Modulo>Endpoints.cs` com método de extensão `Map<Modulo>Endpoints(this IEndpointRouteBuilder)` que cria `MapGroup("/api/<modulo-em-kebab>").WithTags(...).RequireAuthorization(Politicas.X)`. Delegates minimal API recebem serviço/repositório por DI, `IValidator<T>` (FluentValidation, resultado devolvido como `ValidationProblem`) e `CancellationToken`. Retornos tipados (`Results<Ok<T>, NotFound, ValidationProblem, ProblemHttpResult>`) alimentam o OpenAPI. Paginação: `PagedResult<T>(Items, Total, Page, PageSize)`. Erros de negócio: 403/409/422 com `ProblemDetails.detail` = mensagem em PT-BR; `ErroDto` em Empresas.

## DI

- `Application/DependencyInjection.cs` (`AddApplication`): validators por assembly, `TimeProvider`, serviços (`AutenticacaoService`, `PermissoesService`, `UsuariosClientesService`, `TokensIaService`, `ConversaoTrialService`, `ProrrogacaoTrialService`, `PublicacaoAvisoService`, `ConsultaLeituraAvisoService`, `RelatorioEngajamentoService`, `RelatorioUsoFuncionalidadesService`, controle/ação da Fila, `DashboardService`, `ApuracaoIndicadoresService`, `SmarkBiService`, `ModulosEmpresaService`, `AuditoriaService`, `ExportacaoLogsAuditoriaService`) e `*Options` (`BindConfiguration`). Alguns serviços são registrados por factory porque dependem de interfaces implementadas na Infrastructure.
- `Infrastructure/DependencyInjection.cs` (`AddInfrastructure`): resolve a connection string (Parameter Store ou `ConnectionStrings:Default`), registra `AppDbContext` **e** `IDbContextFactory<AppDbContext>` (fábrica usada por consultas paralelas), interceptor de auditoria, repositórios, gateways de tenant, HTTP clients (CrmApi, Queues API, MyBI), S3, Route53 (real só com `Route53:Enabled=true`, senão `LogDnsEmpresa`), `IServicoJobHost` (Windows+habilitado, senão Indisponível) e o worker do dashboard. Registro lê configuração **no startup** (mudar config exige reiniciar).
- Salt do hash legado: `Auth:LegacyPasswordSalt`, com fallback para constante igual à do CRM.

## EF Core sobre tabelas legadas

- Um único `AppDbContext` (`Infrastructure/Persistence/AppDbContext.cs`), `ApplyConfigurationsFromAssembly`; uma `*Configuration` por entidade com `ToTable("<nome legado>")` e nomes de coluna legados (muitos em minúsculas/`idLogin`, `idGrupo`, `dataBaseName`).
- Tabelas de junção legadas (`GruposLogin`, `MenusGrupo`) são property bags. `Menus.path` **não** é mapeado: o acesso por menu usa SQL parametrizado (`AcessoMenuConsulta`).
- `Usuario` (tabela `Logins`) tem **filtros nomeados** `NaoExcluido` (`deleted=0`) e `LoginBackOffice` (`Aplicacao=2 AND ativo=1`); telas de clientes usam `IgnoreQueryFilters`/`IgnoreQueryFilter` seletivo. Atenção ao mexer: afeta login, Permissões, Tokens IA, Dashboard, Trials, Usuários de clientes.
- Sem migrations; testes de persistência criam o schema a partir do modelo (`EnsureCreated`). Mudança de coluna/tabela = script SQL em `Api/Scripts/` + ajuste da Configuration (ver `banco-de-dados.md`).
- Escritas fora do ChangeTracker (`ExecuteUpdate/Delete/Sql`, ADO.NET nos tenants) não passam pelo interceptor de auditoria; há lista fechada em `tests/.../UnitTests/Auditoria/AllowlistEscritasForaDoChangeTracker.cs` e teste `EscritasForaDoChangeTrackerTests` que falha se uma nova escrita desse tipo não for listada/auditada à mão.
- Datas: legado grava `DateTime` em **hora de Brasília** (sem fuso); `RelogioBrasilia` (Application/Common) converte. `AuditoriaEventos` grava UTC (`DataHoraUtc`). Cada módulo documenta qual usa (cuidado com bugs de fuso: ex. fila JobHosts expõe UTC ao front).

## Auditoria centralizada (ADR 0010)

- Tabela `dbo.AuditoriaEventos` (imutável por trigger; expurgo só pela procedure `sp_ExpurgarAuditoriaEventos`); enums com números **imutáveis** (Origem, TipoEvento, AutorTipo, Resultado) compartilhados com o JobHost legado.
- `AuditoriaSaveChangesInterceptor` (Infrastructure/Auditoria): captura Create/Update/Delete das entidades opt-in de `RegistroEntidadesAuditadas` (Usuario, Grupo, Empresa, UsuarioSmarkBi, AbaMyBi, Aviso, Onboard + joins `GruposLogin`, `MenusGrupo`, módulos da empresa, empresas do onboard) **na mesma transação**; falha ao gravar o evento derruba a escrita. Campos sensíveis (senha, hash, `DataBasePassword`, senha BI) saem como `***`. Nunca auditadas: tabelas de log/histórico (`NuncaAuditadas`).
- `IAuditoriaService`: `Registrar` (enfileira no DbContext) e `RegistrarImediatoAsync` (contexto próprio, nunca lança) para login/logout, conversão de trial, publicar/revogar aviso, ações da Fila, exportação, execução de job do dashboard. Contexto do autor: `ContextoAuditoriaHttp` (claims do cookie, IP, user-agent, `X-Correlation-Id` via `CorrelationIdMiddleware`).
- Para auditar uma entidade nova: `RegistroEntidadesAuditadas.Montar()` + rótulos; para escrita fora do ChangeTracker: evento explícito + entrada na allowlist do teste.

## Acesso a tenants (bancos de empresa)

`Infrastructure/Tenant/`: `ResolvedorConexaoTenant` monta a conexão com `Empresas.dataBase*` + `Servidores.nomeServidor` (credencial por empresa guardada em `Empresas.DataBasePassword`; nunca logar). Adaptadores com SQL direto: `NotificacaoCrmSqlServer` (Notificacoes/LoginContador), `LeituraAvisoSqlServer` (leitura), `FilaTenantSqlServer` (`QueueFiles`), `BancoTenantSqlServer` (replicar logins e limpar base na conversão de trial; `Sql/LimparBaseTrial.sql` embutido), `TelefonesUsuarioTenantReader`, `UsuarioBancoMyBi`. Padrão: erro por empresa isolado e sem segredo; paralelismo limitado por config.

## Integrações externas

| Integração | Classe | Config (só nomes) |
|---|---|---|
| API do CRM (PostTrial) | `CrmApi/CriacaoTrialApiHttp` | `CrmApi:BaseUrl`, `TimeoutSegundos` |
| Queues API (e-mail de aviso) + token | `Filas/FilaEmailAvisoHttp` | `QueuesApi:BaseUrl/Endpoint/TimeoutSegundos`; `AuthenticationApi:UrlToken/Usuario/Senha/Subdominio/SenhaEhHash` |
| S3 (imagens de aviso) | `Aws/S3ArmazenamentoImagemAviso` | `Avisos:Bucket/RegiaoAws/AwsProfile/UrlCdn/PastaImagens` |
| Route53 (CNAME na conversão) | `Aws/Route53DnsEmpresa` | `Route53:Enabled/HostedZoneId/Valor/AwsProfile` |
| MyBI | `Integracoes/MyBi/MyBiClient` | `MyBi:UrlBase/Token/Projetos/Dashboard*/Db*/GerenciarUsuarioBanco` |
| Serviço Windows JobHost | `JobHost/ServicoJobHostWindows` | `FilaJobHosts:Servico:*` |
| Parameter Store (SSM) | `Configuracao/ParameterStoreConnectionStrings` | `ParameterStore:*` — **só para a connection string** |

Outras seções de configuração: `Permissoes:IdMenuPaiBackOffice`, `Trials:*`, `RelatorioEngajamento:*`, `Dashboard:*`, `Auditoria:ProxiesConfiaveis`, `Cors:Origins`, `Auth:LegacyPasswordSalt`. Valores: ver `appsettings*.json` (nunca copiar segredos).

## Hospedagem em background

Único `IHostedService`: `HistoricoIndicadoresWorker` (dashboard). Com várias instâncias da API há idempotência por índice único+upsert; o lock do controle do serviço JobHost é em memória (não protege entre instâncias nem contra o 4.8).

## Onde procurar

- Regra de negócio: `Application/<Modulo>/*Service.cs` e `Domain/<Modulo>`.
- Consulta/SQL: `Infrastructure/Persistence/Repositories/<Modulo>Repository.cs`.
- Rota/policy: `Api/Endpoints/<Modulo>Endpoints.cs` + `Program.cs` + `Autorizacao/`.
- Validação de entrada: `Application/<Modulo>/*Validator.cs`.
