# BackOffice.Api, ServicesProxy, serviços Windows e filas
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: como o CRM conversa com o BackOffice (API REST + banco central), o que cada controller da API faz, o que é o WCF legado e como o JobHost consome as filas.

## 1. Visão do relacionamento CRM <-> BackOffice

```
CRM Web (src/SMark/Web) ─┐
SMark.Api (SiteExpress.SMark.Api) ─┼─ SiteExpress.SMark.ServicesProxy (src/SMark/ServicesBackOffice/Api/v1/*Service)
JobHost (App.config) ─┘             │  HTTP + Basic, base = appSetting BackOfficeServicesApiBaseUrl (…/api/v1/)
                                    ▼
                         BackOffice.Api (src/Services/SiteExpress.SMark.BackOffice.Api, local :1999)
                                    ▼
                    BLL/DAL do BackOffice ─► banco BACKOFFICE (Logins, Empresas, Menus, QueueMessages...)

JobHost ─────────────────────────────────► BLL do BackOffice (acesso DIRETO ao BACKOFFICE, sem a Api)
BackOffice MVC ──────────────────────────► BLL direto (não passa pela Api)
```

- O CRM acessa o BACKOFFICE quase só pela API; há um acesso SQL direto residual (`src/SMark/SiteExpress.SMark.DAL/ADO/BaseBancoDados.cs`, connection string `SMarkBackOffice`) e o JobHost usa a BLL direta. Tudo que é identidade, menus/permissões, licenças, empresas, campos dinâmicos, notificações e enfileiramento passa por esta API.
- Nos projetos net9/net10 (`src/SMark/Smark.UI`) a mesma chave existe em `appsettings*.json`; fora do escopo.

## 2. Como o CRM autentica na BackOffice.Api

- Cliente: `ServiceBaseApi` (`src/SMark/ServicesBackOffice/Api/ServiceBaseApi.cs`) monta `Authorization: Basic base64("<subdominio>\<login>:<senha>[:<tipoAplicacao>]")` e faz HTTP sync (`.Result`). Há ctor alternativo que recebe o `hash` (header já pronto). Chamadas ainda gravam log opcional na base da empresa (`ativaLog`).
- Servidor: `[IdentityBasicAuthentication] [Authorize]` em quase todos os controllers (`Filters/BasicAuthenticationAttribute .cs` + `IdentityBasicAuthenticationAttribute .cs`). Quando o usuário vem sem `\`, o subdomínio sai do host da requisição. Se a senha contém `:`, o trecho após o `:` é o `TipoAplicacao`.
- Validação (`Autenticacao/AutenticarUsuario.cs`):
  - tipo `Smark` (padrão): compara usuário/hash em `Logins` (`BLLLogin.RetrieveSingleByLogin`); em `Logins/RealizarLogin` ainda aplica contagem de tentativas (`IsTentativaAcessos`, `MaxTentativaAcessos`, default 5).
  - tipo `Backoffice`: `BLLLogin.RealizarLoginBackOffice`.
  - tipo `API` (contas de serviço): a senha chega **cifrada com AES** (`Util/SMarkCryptography.cs`, chave/IV em appSettings `AES:*`), é decifrada e comparada ao hash.
  - Regex opcional de login (`IsRegexLogin`/`RegexLogin`).
- A sessão do usuário é do CRM (SSO/Redis); a Api só registra `LoginSessions` (id da sessão) em `RealizarLogin` e permite derrubar sessão (`ForceLogoutSession`).
- Empresa do chamador: `BaseController.GetCurrentEmpresa()` extrai o subdomínio do `User.Identity.Name` (`sub\login`).

## 3. Controllers da API (`src/Services/SiteExpress.SMark.BackOffice.Api/v1/Controllers`)

Rotas: `api/v1/{controller}/{action}/{id}` e `api/v1/{controller}/{id}` (`App_Start/WebApiConfig.cs`; atributo `[Route]` só em `UsoFuncionalidades/ConsultaStatusFuncionalidadeBackOffice`). Retorno padrão `HttpResponseMessage` com JSON; erro de negócio vira HTTP 500 com `HttpErroResponseMessage` (`isBusiness`), que o cliente desserializa. Documentação automática em `Areas/HelpPage` (ASP.NET Help Page).

| Controller | Função | Cliente no CRM (`ServicesBackOffice/Api/v1`) |
|---|---|---|
| `Authentication` (**sem** auth) | `SignIn`: valida usuário/senha e devolve token Basic + `IdLogin` + dados de conexão do banco da empresa (nome, usuário, servidor e o campo de senha do banco) | `AuthenticationService` |
| `Logins` (1,3 mil linhas) | CRUD de usuários, logins de replicação (`RetrieveReplicacao`/`UpdateReplicacao`), `RealizarLogin` (CRM), `RealizarLoginMobile`, `Logout`, `ForceLogoutSession`, `AlterarSenhaExpirada`, token de troca de senha (`CriarTokenTrocaSenha`/`RetrieveTokenTrocaSenha`/`InvalidarTokenTrocaSenha`), `ValidarNovaSenha`, blacklist de e-mail, e-mail secundário, `LimparTentativaAcessos` | `LoginService` |
| `Grupos` | Grupos, menus e permissões por grupo (`MenusByGrupo`, `SetPermissaoGrupo`, permissões especiais), validar dependências e excluir | `GrupoService` |
| `Empresas` | `CreateTrial`, `ExisteEmpresa`, `UpdateEmpresa`, `GetOnboard`, `GetEmpresaBySubdominio`, plano do cliente, usuário MyBI, `GetEmpresa(s)` | `EmpresasService` |
| `Licencas` | Solicitações e contagens de licenças (CRM/BI), `InsertLicenciamento` | `LicencasService` |
| `Campos` | Dicionário de campos/entidades (`SmarkEntidades`/`SmarkCampos`/relacionamentos/obrigatórios) | `CamposService` |
| `Notificacoes` | Notificações agendadas por usuário | `NotificacoesService` |
| `Queue` | `Enqueue`: cria `QueueMessage` para o `QueueProcess` do tipo pedido (se o tipo não existir, responde OK com `id=0`, sem erro) | `QueueService` |
| `UsoFuncionalidades`, `RelatorioEngajamento` | Registro de uso/engajamento enviado pelo CRM | `UsoFuncionalidadesService`, `RelatorioEngajamentoService` |
| `CID`, `Cnae`, `Enderecos` | Consultas de apoio (CID, CNAE, IBGE); `Enderecos/AtualizaBaseIBGE` | `CIDService`, `CnaeService`, `EnderecoService` |
| `TokensIA` | Tokens de IA por empresa | `src/SMark/ServicesBackOffice/IA/IAService.cs` |
| `JobHost` (**sem** auth) | `RestartService`: para/inicia o Windows Service do JobHost (nome em appSetting `serviceNameJobHosts`) | sem cliente identificado |

Replicação de usuários: o BackOffice marca `Logins.replicar=1`; o CRM, na `HomeController` da área Home, chama `BLLLogin.ReplicarLogins`, busca `Logins/RetrieveReplicacao`, grava no banco do tenant e confirma por `Logins/UpdateReplicacao`.

Impacto de mudança na API: contratos são DTOs de `src/Global/ServicesDTO` (compartilhados com o CRM). Alterar nome de action, rota ou DTO exige mudar o `*Service` correspondente do CRM **e** a SMark.Api/JobHost/Smark.UI que o usem.

## 4. ServicesProxy do BackOffice (`src/BackOffice/SiteExpress.SMark.BackOffice.ServicesProxy`)

`AWS/SSMService` (Parameter Store; ver `seguranca-e-acesso.md`), `AWS/S3BucketService`, `AWS/Route53Service` (CNAME do subdomínio), `API/MyBIService` (usuário/permissão no BI externo; URL e credencial nos appSettings `MyBIApi:*`), `APIsCore/QueuesAPI` (POST do e-mail de aviso na API de filas externa `QueueCoreAPI:Adress`) e `APIsCore/AuthenticationAPI` (obtém bearer dessa API externa). A API externa de filas/auth ("core") é outro sistema; seu código não está neste escopo.

## 5. WCF legado: BackOfficeServices e BackOfficeServicesControl

- `src/Services/BackOfficeServices` (net4.5): serviço WCF `netTcp` em Windows Service (`BackOfficeServicesWS`), contratos `LoginSvc`, `CamposSvc`, `ConteudoSvc`, `EnderecoSvc`, `LicencaSvc`, `ColaboradorSvc`, `SuporteSvc`, `LogSvc`; base `BackOfficeServiceBase` com autenticação por usuário/senha; endereço base `net.tcp://…:7001/…`. Instalação por `Services/Install_bat/*.bat` (InstallUtil + `sc start BackOfficeServices`).
- `BackOfficeServicesControl` (WinForms net4.5): iniciar/parar/reiniciar o serviço acima.
- Estado: ambos fora de `Smark_SolutionCompleta.sln` e de `BackOffice.sln`; `Tests/BackOfficeServicesTests` só tem `bin/obj` (sem fontes versionadas). O `Web.config` do CRM ainda declara endpoints `netTcp` (Login, Conteudo, Licenca, Endereco, DicionarioDeCampos), mas nenhum `.cs` referencia os proxies WCF (`LoginServiceProxy` etc.) — a Api REST substituiu. Se o serviço ainda roda em produção: (não verificado). Mudanças em BLL/DAL ainda precisam compilar com ele (referencia BLL/Domain/Infra/Ioc).

## 6. JobHost e filas (`src/SiteExpress.SMark.JobHost`, `src/SiteExpress.SMark.Services`)

- Windows Service `JobHostServices` (`JobHost.cs`): `OnStart` -> `WindsorFactory.Iniciar()` -> libera locks órfãos da própria máquina -> laço de ciclos (`Synchronizer.StartProcessAsync`) com intervalo configurável.
- Processos periódicos ligados por appSettings (`isAddEmpresaTrialProcess`, `isRemoveEmpresaTrialProcess`, `isQueueMessageProcess`, notificações de trial 3/5/9/11/15 dias, `BIGocilProcess`...). Limpeza de sessões Redis (`RedisSessionManagerProccess`).
- **Fila** (`QueueMessageProcess`): lê `QueueMessages` pendentes (ordem por `QueuePrioridades.ordem`), reclama em lote com lock (`lockedBy` = máquina, `lockedAt`), mapeia `QueueProcess.classe` -> tipo C# em `SiteExpress.SMark.Services/Process/QueueProcess/*` por reflexão (`Activator.CreateInstance`), executa em paralelo limitado (`IsParallelQueue`, `TotalDeQueuesParalelos`), com timeout por item, retentativas (`QueueMaxTentativas`, padrão 3) e `FinalizarFalha` ao esgotar. Itens duplicados (mesma empresa/login/prioridade/tipo) são serializados.
- `QueueType` (`Global/General/Enum/QueueType.cs`): ContatosDaConta 1, ExportAssistenteContas 3, ExportAssistenteOportunidades 4, RessarcimentoDespesas 5, GestaoDePropostas 6, Georeferenciamento 7, MapeamentoEnderecos 8, Importacao 9, Notificacao 10, ResultadoPorColaborador 11, ContatosEfetuados 12, GeradorDeProposta 13, ExportAnaliseFechamento 14, ImportacaoDeProdutos 15, ExportRelatorioTarefas 16, ExportRelatorioAtividadeRealizada 17, ImportacaoDeBeneficiarios 18, ImportacaoDeComboBox 19, ExportTaxaDeConversao 20.
- **Novo tipo de fila** = (1) valor no enum `QueueType`, (2) classe `Queue...` em `SiteExpress.SMark.Services/Process/QueueProcess` herdando `QueueBaseProcess`, (3) script BACKOFFICE inserindo linha em `QueueProcess` (tipo, `classe`, ativo, prioridade) — exemplo: `Scripts por Sprint/3.32/07 - (BACKOFFICE) Insert-QueueProcess-ExportTaxaDeConversao-MIGRATE.sql`, (4) quem enfileira (CRM -> `QueueService.Enqueue`/`BLLQueueFile.Enqueue`). Sem a linha em `QueueProcess` a Api responde sucesso e nada é enfileirado.
- Saúde: tabela `JobHostHeartbeat` (`UsarHealthCheckHeartbeat`). O MVC mostra a fila em `Ferramentas/ListaJobhosts` e controla o serviço por `ServiceController` local (mesmo servidor).
- Trial: `AddEmpresaTrialProcess` cria empresas-trial pré-provisionadas (scripts em `SiteExpress.SMark.Services/Sql` e na pasta configurada em `pathSql`), `RemoveEmpresaTrialProcess` remove expiradas (SQL de exclusão de empresa/bancos) — ver `banco-de-dados.md`.
