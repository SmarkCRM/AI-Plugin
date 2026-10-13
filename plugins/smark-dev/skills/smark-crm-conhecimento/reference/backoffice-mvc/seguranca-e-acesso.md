# BackOffice 4.8 — autenticação, autorização, perfis e segredos
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: explicar como o login e as permissões funcionam no BackOffice MVC e na BackOffice.Api, e onde há segredos versionados. **Este arquivo não contém valores de segredo, apenas nomes de arquivo/chave.**

## 1. Login do operador no BackOffice MVC

1. `Views/Login/Index.cshtml` + `Content/js/Login.js` chamam **`GET /Login/LogarUsuario?usuario=&senha=`** (usuário e senha vão na querystring; a action não restringe verbo e responde `JsonRequestBehavior.AllowGet`).
2. `LoginController.LogarUsuario` -> `BLLLogin.ValidaLogin(subdominio, usuario, senha, TipoAplicacao.Backoffice)`. O subdomínio é fixo: appSetting `subdominio` (não é digitado). Busca o `Login` ativo, não deletado, com `Grupos.Any()`, da empresa dona do BackOffice.
3. `ValidaLogin`: bloqueio de conta (`ControleBloqueioConta`, 5 tentativas), compara hash, para `Smark` exige `licenciado`, zera contador, avalia expiração de senha (`ControleExpiracaoSenha`, padrão 90 dias, `Empresas.DiasExpiracaoSenha`), atualiza o login.
4. Preenche `IUsuarioLogado` (por sessão), `FormsAuthentication.SetAuthCookie`, e devolve JSON com `menu` (itens de `Menus` com `idPai = BackOfficeIdMenuPai`) e `admin` (primeiro grupo do usuário). O front guarda tudo em `localStorage['dados']`.
5. Troca/forçamento de senha: filtro global `VerificarSenhaExpiradaAttribute` redireciona para `Configuracoes/AlterarSenhaExpirada` quando `SenhaExpirada`, `ForcarTrocaProximoLogin` ou `PrimeiroAcessoSenhaAlterada` (rotas `login`, `error` e `RedefinirSenha*` são isentas). Esse filtro **também é o único que barra anônimos**: sem `IUsuarioLogado.idEmpresa`/`login` redireciona para `Login/Index`.

Política de senha (valores de política, não segredos): mínimo 12 caracteres e classes obrigatórias (`ValidadorSenhaSeguranca`), histórico das últimas 5 (`ControleHistoricoSenhas`, tabela `LoginHistoricoSenhas`), bloqueio após 5 falhas, expiração configurável por empresa, gerador (`GeradorSenhaSegura`). A senha é guardada como **SHA-256 + valor fixo embutido no código** (`Global/General/Functions/Security/Criptografia.cs`, `ToHashSHA256`), sem salt por usuário.

Logins de **WebForms** (`UI/Views/Main/Login.aspx`): usuário no formato `subdominio\login`; sessão em `BackOfficeSession`.

## 2. Autorização (como é de fato)

- `CustomAuthorize` (`MVC/Filters/CustomAuthorizeFilter.cs`, baseado em `Session["UsuarioBackOfficeSession"]`/`MenusBackOfficeSession`) **não está ativo**: `FilterConfig` e os `[CustomAuthorize]` dos controllers estão comentados. As chaves de sessão que ele lê nem são escritas no login atual.
- Na prática: o `Index` de 11 controllers da área Ferramentas faz a checagem "Valida acesso" (o path da requisição precisa estar em um menu de algum grupo do usuário, senão redireciona ao login). As actions `List`, POST, modais e a área `IA`/`Cadastros` e `RelatorioEngajamento`/`RelatorioUsoFuncionalidades`/`Funcionalidades` **não** repetem essa checagem: qualquer usuário autenticado, que conheça a URL, chama a action direto.
- O menu exibido é decidido no navegador (`localStorage`), portanto **esconder menu não é controle de acesso**.
- Ações sensíveis têm checagem pontual: `EmpresasController.IsAdmin()` (`Grupos.admin`) para campos restritos na edição de empresa. Anti-forgery apenas em `ConfiguracoesController`.
- Se o plano mexe em permissão: lembrar que `Permissoes.tipo` (1 adicionar, 2 editar, 3 visualizar, 4 deletar) só é consumido de verdade pelo **CRM** (`CustomAuthorize` do CRM sobre `MenusSessionDTO`), não pelo MVC do BackOffice.

## 3. Perfis e dados de acesso

- `Grupos` (por empresa, campo `Aplicacao`): `admin` (administrador geral do BackOffice; vê "Usuários/Grupos Backoffice"), `adminCrm` (administrador do CRM da empresa), demais grupos (ex.: "Vendedores"). Usuário <-> grupo em `GruposLogin`; grupo <-> menu em `MenusGrupo`; permissões por menu em `Permissoes`; permissões especiais em `GruposPermissoesEspeciais`.
- Usuários do BackOffice = `Logins` com `Aplicacao=2` na empresa do BackOffice; usuários dos clientes = `Aplicacao=1`; contas de serviço/API = tipos 3/4 (ver `TipoAplicacao`).
- Gestão: `Ferramentas/Permissoes` (operadores do BackOffice) e `Ferramentas/GerenciarUsuarios` (clientes). Reset de senha: tela `FormResetSenha` + `GeradorSenha`.

## 4. BackOffice.Api

- Basic auth por requisição (`subdominio\login:senha[:tipo]`), ver `backoffice-api-e-servicos.md`. Credenciais trafegam em Base64 (usar HTTPS; o `BackOfficeServicesApiBaseUrl` local e em alguns ambientes é `http`).
- **Endpoints sem autenticação**: `Authentication/SignIn` (devolve token + dados de conexão do banco da empresa, inclusive campo de senha do banco) e `JobHost/RestartService` (reinicia o Windows Service do JobHost). Controllers que herdam de `ApiController` com `[IdentityBasicAuthentication][Authorize]` são autenticados, mas **a autorização é só "é um usuário válido"**: qualquer usuário autenticado pode chamar qualquer action (inclusive `DeleteLogin`, `DeleteGrupo` por GET).
- `AlterarSenhaExpirada` e fluxo de token de troca de senha existem na Api e são usados pelo CRM.

## 5. Configuração e segredos (somente caminhos)

- **AWS Parameter Store**: `SSMService` lê o JSON de configuração do ambiente (nome vem de `AWSParameterStoreEnvironment`; um valor por desenvolvedor/ambiente). A classe `ConfigurationParametersApplicationSingleton` (`Global/General/ParameterStoreAWS`) expõe `GetConnectString(app="APIBO", db="SMarkBackOfficeContext")` e `GetValue(chave, app="CRM")`. Quando `UsarParameterStore` é falso/ausente, cai no `ConfigurationManager`. No MVC e no WebForms a leitura do SSM ocorre sempre no `Application_Start`.
- **Segredos versionados (caminhos; não copiar valores)**:
  - `src/BackOffice/SiteExpress.SMark.BackOffice.ServicesProxy/AWS/SSMService.cs` e `src/SMark/ServicesBackOffice/AWS/SSMService.cs`: credencial AWS fixa em código.
  - `src/BackOffice/SiteExpress.Smark.BackOffice.MVC.UI/Web.config`, `src/BackOffice/UI/Web.config`, `src/Services/SiteExpress.SMark.BackOffice.Api/Web.config` (+ `Web.aws_hlg.config` em UI e Api), `src/SiteExpress.SMark.JobHost/App.config` (+ `App.aws_dev.config`, `App.aws_hlg.config`), `src/Services/BackOfficeServices/App.config`, `src/BackOffice/DAL/App.config`, `src/BackOffice/Infra/App.config`: chaves AWS, credenciais de e-mail/proxy, Zimbra, MyBI (API e banco), Mongo, AES (chave/IV) e/ou conexões.
  - `src/BackOffice/UI/Views/Migrations/RunMigrations.aspx.cs`: parâmetro fixo de autorização do runner.
  - `src/Global/General/Functions/Security/Criptografia.cs`: valor fixo usado no hash de senha.
  - Tabelas: `Empresas` guarda usuário e senha do banco de cada tenant em texto; `UsuariosSmarkBI` guarda a credencial do MyBI em texto.
- Regras para o agente: nunca ecoar esses valores; não commitar troca de `AWSParameterStoreEnvironment`; ao propor melhorias de segurança, tratar rotação de credencial como atividade de infra (a chave AWS embutida está no histórico do git).
- Telemetria: Application Insights configurado no MVC (`ApplicationInsights.config`).
- Sessão do MVC: sem `sessionState` no `Web.config` do MVC (padrão InProc; o Redis do CRM não é usado pelo BackOffice) — (não verificado em produção/IIS). Reciclagem do pool desloga todos.
