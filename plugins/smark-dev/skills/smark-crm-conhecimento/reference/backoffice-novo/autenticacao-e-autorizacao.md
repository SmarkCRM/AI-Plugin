# BackOffice novo — autenticação, autorização por menu e configuração de acesso

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: como o operador entra, como a sessão funciona, como cada rota é protegida (policies por menu/admin) e como adicionar uma proteção nova. Inclui o Parameter Store (só nomes de chaves, sem valores).

## Login e sessão

- `POST /api/auth/login` (anônimo): valida com FluentValidation, `AutenticacaoService`: busca `Logins` (`UsuarioRepository.ObterPorLoginAsync`, filtros `Aplicacao=2`, `deleted=0`, `ativo=1`), confere `SHA256(senha + salt)` (`LegacySha256PasswordHasher`; salt = `Auth:LegacyPasswordSalt`, fallback constante igual à do CRM) e exige **ao menos um grupo** (`GruposLogin`). Qualquer falha = **401 genérico**; o motivo (`UsuarioInexistente`, `SenhaInvalida`, `SemGrupo`) só vai para a auditoria.
- Sucesso: `SignInAsync` com cookie `smark.backoffice.session`, `HttpOnly`, `SameSite=Lax`, `SecurePolicy=SameAsRequest`, expira em 8 h com `SlidingExpiration`. Claims: `NameIdentifier` (idLogin), `Name` (login), `GivenName` (nome), `Email`, e `Role=Admin` se o login está em grupo `admin=1`. **A claim Role não é usada nas policies** (serve só ao front/`/me`); admin real é consultado no banco por request para a revogação valer na hora.
- A API não redireciona: `OnRedirectToLogin` → 401, `OnRedirectToAccessDenied` → 403.
- `POST /api/auth/logout` audita (quando há sessão) e faz `SignOut`.
- `GET /api/auth/me` (só sessão): `UsuarioAutenticadoDto(id, login, nome, email, isAdmin, telas[])`. `telas` = ids de tela liberados (ver abaixo). Fonte da verdade da sessão no front.
- Mesma origem em produção (SPA servido pela API) → sem token em JS. Em dev, o proxy do Vite mantém o cookie; CORS só para `Cors:Origins` com `AllowCredentials`.
- **Não implementado** vs. 4.8: bloqueio de conta por tentativas, expiração/redefinição de senha, `ForcarTrocaProximoLogin`/`PrimeiroAcessoSenhaAlterada` (gravados pelo CRUD mas não aplicados no login), licenciamento e filtro por subdomínio; **sem SSO** com o 4.8 (quem migrou loga na app nova). Colunas `ContaBloqueada`, `TentativasLoginFalhadas` existem em `Logins` e são zeradas só no desbloqueio manual de Usuários de clientes.

## Autorização por menu (mecanismo central)

Replica a regra de visibilidade de menu do 4.8: o login precisa ter, por algum grupo (`Logins → GruposLogin → MenusGrupo → Menus`), um menu cujo `path` (sem `..`) **contém** (ou, para menus da app nova, **é igual a**) um caminho conhecido.

- `Api/Autorizacao/AcessoMenu.cs`: `AcessoMenuRequirement(CaminhoMenu[] caminhos, bool permitirAdministrador)` com constantes dos paths e listas por tela (`TrialsMenu`, `AvisosMenu`, `T1/T2/T3`...). `AcessoMenuHandler` (Scoped): memoiza por request a consulta; se `PermitirAdministrador` e o login é admin, passa direto. `AcessoMenuNegadoResultHandler`: falha de menu → **403 ProblemDetails** com `detail` = "Você não tem permissão para acessar esta tela."; 403 da policy `LogsAuditoria` também é auditado.
- `Infrastructure/Autorizacao/AcessoMenuConsulta.cs`: SQL parametrizado com `CHARINDEX` ("contém") ou igualdade (`Exato`), exigindo `Logins.idEmpresa > 0` e `deleted = 0`. Comparação na collation do banco (case/acento-insensível), diferente do `Contains` ordinal do 4.8.
- Sempre path **completo** (ex. `/Ferramentas/Empresas`); substring solta casaria `../IA/ControleTokensIaPorEmpresas`.
- Sem claim de menu no cookie: revogação vale na hora; custo de ≤ 1 consulta por lista de caminhos por request.
- `AdministradorBackOfficeRequirement/Handler`: login em grupo `Grupos.admin=1` (consulta `IPermissoesRepository.EhAdministradorAsync` por request).

## Policies (registradas inline em `Program.cs`; nomes em `Politicas.cs`)

| Policy | Regra | Usada em |
|---|---|---|
| `AdministradorBackOffice` | admin | `/api/permissoes/*`, `GET /api/empresas/servidores`, `POST /api/dashboard/historico/reprocessar` |
| `TokensIa`, `TokensIaPorUsuarios`, `TokensIaPorEmpresas` | menu antigo `../IA/...` (contém) **ou** novo `/ia/tokens*` (exato) | `/api/tokens-ia/{chamadas,por-usuario,por-empresa}` |
| `TokensIaQualquerTela` | qualquer um dos 3 | `/api/tokens-ia/opcoes*` |
| `Avisos` | menu `/Ferramentas/Avisos` | todo `/api/avisos` (inclui leituras) |
| `Empresas` / `EmpresasEditar` | menu `/Ferramentas/Empresas` (lista própria p/ editar) | `/api/empresas` / escrita |
| `EmpresasMyBiAdmin` | menu Empresas **E** admin | MyBI global (`sincronizaveis`, `abas`) |
| `Trials`, `TrialsCriar`, `TrialsConverter`, `TrialsProrrogar` | todas = menu `/Ferramentas/Trials` | `/api/trials/*` (separação por ação ainda não existe) |
| `RelatorioEngajamento` | menu `/Ferramentas/RelatorioEngajamento`, admin também passa | `/api/relatorio-engajamento` |
| `RelatorioUsoFuncionalidades` | menu `/Ferramentas/RelatorioUsoFuncionalidades`, admin passa | idem |
| `AuditoriaCnpj` | menu `/Ferramentas/Auditoria` | `/api/auditoria-cnpj` |
| `Onboards` | menu `/Ferramentas/Onboards` | `/api/onboards` |
| `FilaJobHosts` / `FilaJobHostsControle` | menu `/Ferramentas/ListaJobhosts` / menu **E** admin | consulta e ações por item / `/servico/{iniciar,parar,reiniciar}` |
| `LogsAuditoria` | menu **exato** `/ferramentas/logs-auditoria`, sem bypass de admin | `/api/logs-auditoria` |
| `UsuariosClientes` | menu `/Ferramentas/GerenciarUsuarios`, sem bypass | `/api/usuarios-clientes` |
| (só sessão) | `RequireAuthorization()` | `/api/auth/me`, `/api/dashboard/{contadores,empresas-ativadas,trials-ativos,historico}` — lista explícita em `CoberturaDeAutorizacaoTests` |

Regra do teste de arquitetura: todo `/api/*` precisa de policy/role, exceto login/logout e a lista "SoLogin"; incluir rota nova nessa lista é decisão de acesso consciente.

## `telas` em `/api/auth/me`

`Api/Autorizacao/TelasLiberadas.cs`: lista `(id, caminhos)` avaliada com `AcessoMenuConsulta` (1 consulta por tela) e regras extras: `relatorio-engajamento` para admin; `fila-jobhosts-controle` (menu + admin); `empresas-mybi-admin` (editar + admin). Ids: `tokens-ia`, `tokens-ia-usuarios`, `tokens-ia-empresas`, `trials`, `trials-criar`, `trials-converter`, `trials-prorrogar`, `auditoria-cnpj`, `onboards`, `fila-jobhosts`, `fila-jobhosts-controle`, `logs-auditoria`, `relatorio-uso-funcionalidades`, `avisos`, `empresas`, `relatorio-engajamento`, `empresas-editar`, `empresas-mybi-admin`, `usuarios-clientes`. Observação: o admin que passa por `PermitirAdministrador` em Uso Funcionalidades **não** recebe o id `relatorio-uso-funcionalidades` se não tiver o menu (só Engajamento tem a regra extra) — (diferença policy x telas; não verificado em execução).

## Como proteger uma rota/tela nova (checklist)

1. Definir o path do menu (existente no 4.8 → "contém" com path completo; novo → script `INSERT` em `Menus`/`MenusGrupo` + `Exato`).
2. `AcessoMenu.cs`: constante + lista `CaminhoMenu[]`.
3. `Politicas.cs`: nome; `Program.cs`: `AddPolicy(...)` com `AcessoMenuRequirement` (e `AdministradorBackOfficeRequirement` se menu E admin).
4. Endpoint: `RequireAuthorization(Politicas.X)` no grupo.
5. `TelasLiberadas.cs`: id da tela (e regra extra, se houver).
6. Front: `NAV_ITEMS` + `exigirAcesso({telas:[id]})`.
7. Testes: 200/401/403 com `FakeAcessoMenuConsulta`; `CoberturaDeAutorizacaoTests` deve continuar verde; teste de `TelasLiberadas*`.
8. Deploy: o menu precisa existir e estar vinculado aos grupos nos bancos de hlg/prd **antes** (senão 403 para todos, ou só admin onde houver bypass).

## Parameter Store e configuração de acesso (sem valores)

- `ParameterStore:Enabled` (+ `EnvironmentParameterName`, `AwsProfile`, `Region`, `Application` = `APIBO`, `DatabaseConnectionName` = `SMarkBackOfficeContext`): lê um parâmetro SSM com JSON `parameters[].application → configs.dataBaseConnections[]` e usa a connection string (normalizada com `TrustServerCertificate` se não houver Encrypt). Credencial AWS = profile do `~/.aws/credentials` (ex.: sessão MFA) ou cadeia padrão do SDK. **Só a connection string vem do Parameter Store**; demais seções (`CrmApi`, `QueuesApi`, `AuthenticationApi`, `MyBi`, `Route53`, `Avisos`) vêm de appsettings/variáveis de ambiente — de onde vêm em hlg/prd: (não verificado).
- Desligado: `ConnectionStrings:Default` (testes/CI/banco local).
- `appsettings.Development.json` é **versionado** e já traz `ParameterStore:Enabled=true` apontando a um parâmetro de dev individual; o README orienta `git update-index --skip-worktree`. Cada dev deve usar o seu parâmetro.
- Segredos: `AuthenticationApi:Senha`, `MyBi:Token/DbPassword` etc. devem vir de Parameter Store/ambiente — hoje o código só os lê de `IConfiguration`.
