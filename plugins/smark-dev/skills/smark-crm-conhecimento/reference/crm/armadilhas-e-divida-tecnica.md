> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Armadilhas, dívida técnica e suspeitas

Objetivo: lista de pegadinhas que costumam gerar bug ou retrabalho. Cada item traz o selo `[V]` (conferido no código nesta data) ou `[NV]` (não verificado). Nenhum valor de segredo é citado; só caminhos.

## Segredos versionados (só caminhos; nunca copiar valores)
`[V]` contêm chave AWS (`AKIA...`) e/ou senhas/secrets em texto puro:
- `src/SMark/ServicesBackOffice/AWS/SSMService.cs` e `src/BackOffice/SiteExpress.SMark.BackOffice.ServicesProxy/AWS/SSMService.cs` (literais nos campos `AccessKeyId`/`SecretAccessKey`; o construtor padrão os usa).
- `src/SMark/Web/Web.config`, `Web.aws_dev.config`, `Web.aws_hlg.config`; `src/Services/SiteExpress.SMark.BackOffice.Api/Web.config`; `src/SMark/SiteExpress.SMark.Api/Web.config`; `src/BackOffice/UI/Web.config`; `src/BackOffice/SiteExpress.Smark.BackOffice.MVC.UI/Web.config`.
- `src/SiteExpress.SMark.JobHost/App.config`, `App.aws_dev.config`, `App.aws_hlg.config`; `src/SiteExpress.SMark.JobHost.Console.TestGeradorProposta/App.config`; `src/SiteExpress.Smark.Migracao.File/App.config`; `src/Consoles/.../ProdutosImgToS3/App.config` e `.../UploadsToS3/App.config`; `src/SMark/Smark.UI/appsettings.json`.
- No `src/SMark/Web/Web.config` há valores preenchidos para chaves como `AWSAccessKeyId`, `AWSSecretAccessKey`, `Proxy_Password`, `aes:key`, `aes:iv`, `Becon:apiKey`, `APICONNECTINC:password`, `Protheus:password`, `Solus:Integracao*`, `ZimbraAdminSenha`, `UserSenhaTrial`, `UserSenhaIntegracao`, `BackOfficeServicePassword`, `HashBKUser`, `*client_secret`, `serilog:write-to:Seq.apiKey`. Os configs aws_dev/aws_hlg também repetem chaves AWS e `HashBKUser`.
- `[V]` arquivo de credencial OAuth versionado: `src/SMark/SiteExpress.SMark.Api/IntegracaoGmail/credentials.json`. Envs Docker: `infra/smark-ui/docker/env/.env.smark-ui.*` (conteúdo não inspecionado, `[NV]`).
- `[NV]` `.ai-logs/` tem 16 transcrições `.jsonl` rastreadas no git apesar do `.gitignore`; não foram abertas e podem conter segredos.
- `[NV]` padrões `password=`/`pwd=` aparecem em muitos `.cs`/`.config` do BackOffice, consoles e DBDeployer (a maioria pode ser nome de campo; checar caso a caso).
- O hook `src/.githooks/pre-commit` só protege se instalado (`git config core.hooksPath .githooks`); os configs versionados trazem `AWSParameterStoreEnvironment=env-localPedro` (UI), `env-localCassio` (JobHost), `env-localRegis` (Api legada). Não commitar a troca por dev.
- `RELATORIO_ALTERACOES.md` diz que as credenciais do `SSMService` foram removidas e que `DbConfiguration` usa `Encrypt=True`; o código atual **não** reflete isso `[V]`.

## Segurança (suspeitas confirmadas)
- `[V]` Logout do legado não invalida o SSO: `HomeController` apaga `sso:{token}` sem o prefixo `Redis:ApplicationName` (que é `Smark.UI` no config e é usado na leitura e na gravação).
- `[V]` `isAdmin`/`isAdminCRM`/`supervisionados` vêm de cookies cujo valor é Base64 de texto (`CookieWrappers`): editável pelo usuário.
- `[V]` `CustomAuthorize`: casamento por substring (`menu.path.ToUpper().Contains(CONTROLLER)`) e downgrade para Visualizar por `Contains` em `INDEX/LIST/PESQUISAR/GET/PAGINAR` no nome da action; atributo guarda estado (`AccessLevel`, `Controller`) na instância. Ação com nome tipo `...Get...` ou `...List...` marcada como Deletar/Editar vira Visualizar.
- `[V]` `JobHostController.RestartService` e `AuthenticationController.SignIn` da BackOffice.Api sem autenticação.
- `[V]` `/Integracao/RDStation/AddOpportunity` liberado dentro do `CustomAuthorize`; cerca de 20 ocorrências de `[AllowAnonymous]` em webhooks/callbacks de `Areas/Integracao`.
- `[V]` Busca livre concatenada em `LIKE '%...%'` no SQL de `DALOportunidades.montarfiltro` (ver `banco-de-dados.md`).
- `[V]` `DbConfiguration` monta conexão com `Encrypt=False`.
- `[NV]` controllers de teste/diagnóstico em produção: `Relatorios/TesteController`, `Home/TestRDStationController`, `Areas/Integracao/WebServices/Teste.asmx`, `Scripts/Teste.js`.

## Front-end e build
- `[V]` `Scripts/master.js` não parseia (linha 337); `master.min.js` antigo funciona; 14 views carregam `master.js` cru.
- `[V]` `applicationmin.min.css` tem tokens `--azul` que o fonte não tem; regenerar apaga.
- `[V]` Não há regra única `.js` x `.min.js`: 191 referências a `.js` e 319 a `.min.js` nas views (ex.: `OportunidadesCards/Index.cshtml` carrega `OportunidadesCards.js`).
- `[V]` `MvcBuildViews=false`; erro de Razor só em runtime. UI csproj antigo exige `Compile/Content Include`.
- `[V]` `StaticFileCache.Version` muda a querystring a cada hora (usa `DateTime.Now.Hour`), além do timestamp do arquivo.
- `[V]` Chave em `Mensagens.resx`/`Label.resx` sem atualizar `*.Designer.cs` só funciona por string; mensagens com chave inexistente aparecem como a própria chave.

## Backend e dados
- `[V]` `BaseController.BeginExecute` engole exceção de `InicializarPermissoes`; cultura fixa `pt-BR` (comentário diz que viria da Empresa).
- `[V]` `BLLBase.Update(entidade, bool validar = false)` não valida por padrão; `AddSemValidacao`/`UpdateSemValidacao` também não.
- `[V]` `DALBase<T>.CountAsync(expressao)` ignora a expressão; `RetrievePaginate(expressao, pageSize, pageCount)` lança `NotImplementedException`; `Delete(int)` é exclusão física.
- `[V]` `AutoDetectChangesEnabled=false` e `ValidateOnSaveEnabled=false` no `SMarkContext`: sem `Update` explícito não persiste; DataAnnotations não validam no save.
- `[V]` Dois containers Windsor (web x thread): `Task.Run`/Hangfire/JobHost resolvem no `ContainerThread` (`PerThread`); `Resolver<T>()` retorna `null` se não iniciado.
- `[V]` Filtro de pesquisa novo precisa estar em todos os métodos que montam `OportunidadeSearchVMDTO` e na `montarfiltro`, senão lista, kanban, count e total divergem.
- `[V]` `RunMigrations` só lê `Sql/Migrations`; `Scripts por Sprint` não passa por ele; ordem lexical; `GO` quebra; 61 scripts antigos têm `GO`. Como os scripts de sprint são aplicados: `[NV]`.
- `[V]` `Migrations/` parou em 2021; a prática atual é `Scripts por Sprint` (3.33 em out/2026). Documentos antigos (`doc/guides/MIGRATIONS.txt`, `MERGE.txt`, `CHANGESET.txt`, `.docx`) são da era TFS e mencionam `trunk`.
- `[V]` Código morto: `SiteExpress.SMark.Ioc.Unity`; contextos Mongo (`SMarkMongoDBContexto`, `SMarkCountContexto`, `SMarkLogContexto`) com consumidores comentados; `ActionLogFilter` vazio; `HorarioDeAcessoFilter` sem uso; `RedisSessionManagerProccess` nunca iniciado; chaves Mongo/Zimbra no config.
- `[V]` `SiteExpress.SMark.UI.Tests` e `UI.Functional.Tests` (net4.5) estão fora da solution; o teste ativo é `src/Tests/SiteExpress.SMark.Tests` (mistura xunit, MSTest e Playwright.MSTest; `MSTestSettings.cs` ativa paralelismo por método). Cobertura é só de regras puras da BLL e de JS; controllers/DAL/views não têm teste.
- `[V]` Projeto `SiteExpress.SMark.BLL` (net48) referencia o projeto web `BackOffice/UI` por causa de `UIHelper`/`Util`; mudanças no BackOffice.UI podem quebrar o build da BLL.
- `[V]` `JobHost` chama o Parameter Store mesmo com `UsarParameterStore=false` e em falha segue "degradado".
- `[V]` Pasta do projeto `JobHost.Process` chama-se `SiteExpress.SMark.Services` (nome diferente do csproj/namespace `SiteExpress.SMark.JobHost.Process`).
- `[NV]` `SSMService()` sem argumentos consulta o metadata EC2 (169.254.169.254) para a região com `HttpClient` padrão; fora da AWS pode demorar até o fallback.
- `[NV]` Chaves repetidas em `Web.config`/`App.config` (ex.: `TempoSessaoUsuarioAberta`, `MongodbConnectionString`, `BackOfficeServicesApiBaseUrl`), provavelmente em blocos comentados.
- `[NV]` `RegisterTypesConfig` x `ContainerBootstrapper/WindsorActivator`: dois pontos de bootstrap Windsor.

## Convenções que enganam
- Grafias do código: `MensagemTipo.Sucess`, `#divMsgSucess`, `Conta.carteiraExcusiva`, `QueueExportGestaoPropopstas`, `dataBasePassord`.
- Pasta `BancodeDados` (BLL/DAL) x Area `BancoDeDados`; `Integracao` (BLL) x `Integracoes` (DAL).
- "Oportunidade" e "pós-venda" são a mesma tabela (`tipo`): regra da BLL pode valer para os dois.
- "Painel de oportunidades" = `OportunidadesCards`, não `Oportunidades`.
- Texto de submenu vem de `Menus.menu` no BACKOFFICE e só muda após novo login.
- `Smark.*` (net9/net10) não é o alvo; `src/BackOffice` é de outro agente.
