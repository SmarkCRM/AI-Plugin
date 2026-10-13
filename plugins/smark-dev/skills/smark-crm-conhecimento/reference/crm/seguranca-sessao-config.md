> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Segurança, sessão, SSO e configuração (sem valores)

Objetivo: explicar login, sessão, autorização por menu, SSO via Redis e as duas fontes de configuração (Parameter Store x tabela `Configuracoes`), com as pegadinhas já confirmadas no código. Nenhum valor de segredo aparece aqui; os arquivos citados contêm segredos versionados.

## Login e sessão
- `Controllers/LoginController.cs` (`Login` POST): captcha opcional (`IsCaptcha`, `CountRegexLogin`, chave `CaptchaKey` do Parameter Store), `BLLLoginDTO.RealizarLogin` (BLL/Services/Login) -> `LoginService` (ServicesProxy) -> BackOffice.Api `Logins/RealizarLogin`. A resposta vira `LoginSessionDTO` (empresa, grupos, permissões especiais, módulos) e `MenusSession`/`MenusCards` na **Session**, que é **Redis** (`system.web/sessionState` modo Custom, provider `RedisSessionProvider`, timeout 240 min). O `LoginController` legado **não grava** o cookie `SMSSO`: quem grava cookie e payload SSO é o `Smark.UI` (`Response.Cookies.Append("SMSSO", ...)`); o legado só lê e restaura a sessão (ver SSO abaixo).
- Cookies de apoio: `CookieWrappers` guarda `isAdmin`, `isAdminCRM`, `supervisionados`, abas, etc. O nome do cookie usa subdomínio+idLogin "encriptados" e o valor é só Base64 de texto puro (ex.: `true`/`false`). Ver risco abaixo.
- Política de senha: `BLL/Seguranca` (`ControleExpiracaoSenha`, `GeradorSenhaSegura`, `ValidadorSenhaSeguranca`), `VerificarSenhaExpiradaAttribute` (global), configs `ControleForcaSenhaAtivo`/`DiasForcarTrocaSenha` (tabela `Configuracoes`), `TrocarSenhaController` (rota com `{hash}`). Acesso único: `AcessoUnicoFilter` compara `identificadorDeAcesso` da sessão com o do banco e redireciona para `Home/Logout`.
- Subdomínio: `UsarSubDominio` (false no config versionado) e `DominioUrl`; com `true`, `CustomAuthorize` nega se o subdomínio do host diferir da empresa.

## Autorização (`Web/Filters/CustomAuthorizeFilter.cs`, classe `CustomAuthorize`)
1. Exceção fixa: `/Integracao/RDStation/AddOpportunity` passa sem autenticação.
2. Sessão vazia -> tenta restaurar do SSO (cookie `SMSSO` -> Redis hash `<Redis:ApplicationName>sso:<token>`, campo `data`, JSON `SsoSessionPayload` {Id, Login, SubDominio, UsuarioLogado, Menus, ExpiresUtc}); expira por `ExpiresUtc`. A mesma lógica existe em `Global.asax.cs` (`Application_AcquireRequestState` -> `TryRestoreSessionFromSso`).
3. Exige cookie admin **ou** `SMSSO`.
4. `AccessLevel == NaoDefinido` (atributo só na classe) = apenas logado. Com `AccessLevel`, procura nos menus da sessão (`MenusSessionDTO`, pai e filhos) um `path` que **contenha o nome do controller** (comparação por substring em maiúsculas) e checa se a permissão do tipo existe.
5. Se o nome da action contiver `INDEX`, `LIST`, `PESQUISAR`, `GET` ou `PAGINAR`, o `AccessLevel` é **sobrescrito para Visualizar** (por `Contains`, não por igualdade; atinge nomes como `TARGET`, `...List...`, e a atribuição fica gravada na instância do atributo).
6. Não autorizado -> `HandleUnauthorizedRequest`: se a sessão caiu, tenta `ForceLogoutSession` na BackOffice.Api (usa `HashBKUser`), grava `LogSystem` (`LogoutAuto`) e redireciona para `Login/SessionExpiredLogin`.
- Convenção: `[CustomAuthorize]` na classe e `[CustomAuthorize(AccessLevel = TipoPermissao.X)]` nas actions. `[AllowAnonymous]` só em Login, Cadastro, TrocarSenha, AceiteBasesLegais, DeclaracaoDeSaude, DocumentosBeneficiario, Download, Redirect, EmailsDoSistema, Configuracoes (raiz) e nos callbacks de integração.
- Níveis de permissão: (1) CRUD por tela (`TipoPermissao` Adicionar 1/Editar 2/Visualizar 3/Deletar 4), (2) permissão especial do grupo (`BLLLoginDTO.VerificarPermissoesEspeciais(TipoPermissaoEspecial.X)`), (3) admin (`isAdmin`/`isAdminCRM`/`BLLLoginDTO.IsAdmCrm`), (4) módulo contratado (`ModulosDoSistema`, `VerificaPermissaoModulo`). Tela nova exige `Menus` no BACKOFFICE (ver `banco-de-dados.md`).

## Endpoints anônimos de integração (webhooks/callbacks)
`Asten/ClickSign/D4Sign/DocuSign` `PostEventNotificationAsync`, `D4Sign GetSafesAsync`, `Dinamize` (`ReceberLeadDinamizeAsync`, callback OAuth, `PaginaRetorno`), `DocuSign/Zoom GetAuthorizationCallbackAsync`, `Generico PostGenericoRequestAsync`, `Google` callback, `RDStation` (`AddOpportunity`, `RegisterCallbackAsync`, `ContactHasMarkedOpportunityAsync`), `Reev PostReevRequestAsync`, `Zenvia` (`CallBackNovaMensagem`, `CallBackStatusMensagem`). Qualquer mudança neles precisa considerar que não há sessão.

## BackOffice.Api (autenticação)
- Web API 2 em `Services/SiteExpress.SMark.BackOffice.Api`. Controllers com `[IdentityBasicAuthentication]` + `[Authorize]`: header Basic = base64(`subdominio\login:senha[:tipoAplicacao]`), validado por `AutenticarUsuario.Validate`. A senha da sessão do usuário (`senhaWebApi`) é reutilizada pelo CRM em todas as chamadas.
- **`AuthenticationController.SignIn` e `JobHostController` não têm atributo de autenticação.** `JobHostController.RestartService` (GET) para e reinicia o serviço Windows definido em `serviceNameJobHosts` sem autenticação (verificado). `SignIn` devolve token e dados de conexão do banco do tenant.

## SSO compartilhado com o Smark.UI (net10)
- Contrato: cookie `SMSSO` + chave Redis `{Redis:ApplicationName}sso:{token}` (campo `data`). O Smark.UI grava via `IDistributedCache` (`$"sso:{token}"`; `InstanceName` do cache = `Redis:ApplicationName`, concatenado direto antes de `sso:`); o legado lê com o mesmo prefixo. No `Web.config` versionado `Redis:ApplicationName` não é vazio.
- Mudança em `LoginSessionDTO`/`MenusSessionDTO` (Global.ServicesDTO.Session) ou na chave impacta os dois lados.
- **Divergência verificada**: o logout legado (`Areas/Home/Controllers/HomeController.cs`, ~linha 970) apaga `sso:{token}` **sem** o prefixo `Redis:ApplicationName` que a leitura usa. Como o prefixo é não vazio, o `KeyDelete` não encontra a chave: o SSO continua válido no Redis até `ExpiresUtc` (o cookie `SMSSO` é expirado no browser). O `catch {}` vazio em volta do `KeyDelete` esconde a falha; o logout do `Smark.UI` (`Areas/Home/Controllers/HomeController.cs`, `_distributedCache.RemoveAsync`) usa o prefixo certo, então só o logout legado está quebrado.
- **Domínio do cookie (hipótese, valores de ambiente não comparados):** o logout legado expira o `SMSSO` com `Domain = AppSettings["DominioUrl"]`; o `Smark.UI` grava com `Domain = Auth:CookieDomain` (`Smark.UI/Controllers/LoginController.cs`, ~linhas 864-880). Se os dois valores diferirem num ambiente, o browser trata como cookies distintos e o original sobrevive ao logout; combinado com o SSO vivo no Redis, a sessão "ressuscita". Confira os dois valores por ambiente (sem copiá-los para tickets).
- `Startup.cs` (OWIN): SignalR com backplane Redis, host/porta/applicationName lidos do provider de sessão.

## Configuração: duas fontes
| | Parameter Store (plataforma) | Tabela `Configuracoes` (tenant) |
|---|---|---|
| Acesso | `ConfigurationParametersApplicationSingleton.GetValue(key, app="CRM")`, `GetConnectString(app="APIBO", db="SMarkBackOfficeContext")`, `GetServidor(idServidor)` | `BLLConfiguracao.RetrieveByNome(NomeConfiguracao.X)` (enum em `Domain/Enumeradores/Enumeradores.cs`, a partir da linha 389) |
| Escopo | Todos os clientes do ambiente | Uma empresa (banco do tenant), editável em `Configuracoes/DeSistema` |
| Fallback | Valor vazio/ausente no Parameter Store -> `AppSettings.Get(key)` do `Web.config`/`App.config` | — |
- Carga: `Global.asax.IniciarParameterStore()` (UI e BackOffice.Api: só se `UsarParameterStore=true`) usa `SSMService` com o nome do parâmetro em `AWSParameterStoreEnvironment`. JSON: `parameters[] { application, configs { servidoresDeBanco[], dataBaseConnections[], keys[] } }`; aplicações usadas: `CRM`, `APIBO`. O **JobHost chama o SSM sem checar `UsarParameterStore`** e, se falhar, só loga e segue com a configuração local.
- Chaves lidas via singleton (nomes): `HashBKUser` (47 usos), `GoogleKeyApiConta`, `KeyGoogleMapsApi`, `UrlGoogleMapsApi`, `CaptchaKey`, `aes:key`/`aes:iv`, `AWSAccessKeyId`/`AWSSecretAccessKey`/`AWSBucketName`/`AWSS3ServiceUrl`, `Proxy_UserName`/`Proxy_Password`, `Send_Mail_Host_SMTP`, `serilog:write-to:Seq.apiKey`, `rdstation:*`, `docusign:*`, `Microsoft:OFFICE365:*`, `Becon:apiKey`, `APICONNECTINC:*`. A conexão do Hangfire é `GetConnectString("CRM","HangfireConnection")`.
- Chaves do `Web.config` (por nome; muitas com URLs por integração): feature flags `Is*` (`IsHangfire`, `IsAcessoUnicoFilter`, `IsCspFilter`, `IsCountFilter`, `IsLogSystem`, `IsCaptcha`, `IsValidateAntiForgeryBySession`, `IsSetMinThreads`), `Redis:*`, `BackOfficeServicesApiBaseUrl`, `BackOfficeServiceUser/Password`, `QueueCoreAPI:Adress`, `EmailIntegrationAPI:*`, `MarkAPI:Url`, `AuthenticationAPI:AdressToken`, URLs/ids de RDStation, DocuSign, D4Sign, ClickSign, Asten, Dinamize, Zenvia, Zoom, Solus, Protheus, Becon, Unimed CBS, Movidesk, Mongo (sem uso efetivo) e Zimbra (legado).
- Para valor novo de plataforma: ler com `GetValue` (e a chave deve existir também no config base, senão o fallback devolve `null`) e cadastrar no Parameter Store de cada ambiente. Valor que o cliente ajusta: novo `NomeConfiguracao` + script `(EMPRESAS) Insert-Configuracoes-...` + tela em `DeSistema`.
- Transforms: `Web.aws_dev.config`/`Web.aws_hlg.config` (UI, BackOffice.Api, Api legada) e `App.aws_*.config` (JobHost); `Web.Release.config` só remove `debug`.

## Riscos de segurança confirmados (resumo; caminhos em `armadilhas-e-divida-tecnica.md`)
- Segredos versionados (AWS, senhas, client secrets) em vários configs e em `SSMService.cs` (duas cópias).
- `isAdmin`/`isAdminCRM`/`supervisionados` decididos por cookie editável (Base64 simples).
- Busca livre concatenada no SQL de `DALOportunidades` (ver `banco-de-dados.md`).
- Endpoints sem autenticação na BackOffice.Api; logout que não invalida o SSO.
- Pre-commit hook (`src/.githooks/pre-commit`) bloqueia `AKIA...` e `password=`/`pwd=` apenas se instalado (`git config core.hooksPath .githooks`).
