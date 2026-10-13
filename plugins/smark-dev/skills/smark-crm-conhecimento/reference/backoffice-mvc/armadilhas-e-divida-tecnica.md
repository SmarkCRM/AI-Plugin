# BackOffice 4.8 — armadilhas e dívida técnica visíveis
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: lista de pegadinhas reais encontradas no código, para o agente alertar antes de planejar correção ou melhoria. Itens de segurança estão resumidos aqui e detalhados em `seguranca-e-acesso.md`. Nenhum valor de segredo é citado.

## 1. Impacto cruzado (o que mais quebra)

- **Código compartilhado**: BLL/DAL/Domain/Infra/Ioc/ServicesProxy(BO) são usados por BackOffice MVC, BackOffice.Api, JobHost, WebForms, WCF legado e projetos net10 (alvo `net10.0`). Mudar assinatura de método de BLL/DAL ou de entidade EF exige varrer todos esses hosts (`Grep` por `new BLLxxx` em `src/`).
- **Contratos com o CRM**: DTOs em `src/Global/ServicesDTO` e rotas `api/v1/...` são consumidos por `src/SMark/ServicesBackOffice/Api/v1/*Service`. Renomear action/DTO sem alterar o cliente quebra login e menus do CRM.
- **Enums duplicados por valor**: `ModulosDoSistema` mora no CRM (`SiteExpress.SMark.Domain`) e `ModulosSistema.enumModulo` (BACKOFFICE) guarda o número; `QueueType` (Global) e `QueueProcess.type`/`classe` idem; `TipoAplicacao` e `Menus` (Global). Alterar enum sem script/dados correspondentes dessincroniza.
- **Menus por nome**: o CRM compara `Menus.menu` por texto em lugares fixos (`SMark/Web/Controllers/BaseController.cs`, `LoginController.cs`, `Models/MenuPermissoesVM.cs`, `TarefasController`: "Relatórios", "Paineis", "Gerenciar Tarefas", "Assistente", "Acompanhamento de Relatórios"...). Renomear linha de `Menus` pode desligar comportamento no CRM; o texto exibido vem do dado. Usuários logados só veem a mudança após novo login (menus ficam na sessão).
- **Tenants**: script `(EMPRESAS)` roda em **todos** os bancos de clientes; `Empresas`/`Logins` central e tabelas do tenant são universos diferentes (ex.: `Notificacoes`, `QueueFiles`, `LoginContador` são do tenant).
- **Dois ServicesProxy e dois `SSMService`** (CRM e BackOffice): correção em um deve ser replicada ou justificada.
- **Fila sem `QueueProcess`**: `Queue/Enqueue` responde OK (`id=0`) sem enfileirar quando não existe o tipo.

## 2. Autenticação/autorização (resumo)

- Login do MVC por **GET** com usuário e senha na querystring (`/Login/LogarUsuario`); credenciais podem aparecer em log de IIS/proxy/histórico.
- Autorização do MVC praticamente só no `Index` de 11 controllers; `CustomAuthorize` comentado; actions JSON/POST e áreas `IA`/`Cadastros` abertas a qualquer usuário autenticado; menu escondido só no navegador.
- Sem `[ValidateAntiForgeryToken]` fora de `ConfiguracoesController` (CSRF em exclusões, inativação etc.).
- Api: `Authentication/SignIn` e `JobHost/RestartService` sem autenticação; `DeleteLogin`/`DeleteGrupo`/`ForceLogoutSession` via **GET**; autorização só "usuário válido". `SignIn` devolve dados de conexão do banco da empresa.
- Senha: SHA-256 com valor fixo no código; senha do banco de cada tenant e do MyBI em texto nas tabelas `Empresas`/`UsuariosSmarkBI`.
- `AvisosController.AutenticarAPIQueues` usa o hash da senha do operador (`IsHash`) para autenticar na API externa de filas.
- Segredos versionados em Web.config/App.config/`SSMService.cs`/`RunMigrations.aspx.cs`/`Criptografia.cs` (lista em `seguranca-e-acesso.md`).

## 3. Estado e ciclo de vida

- `IUsuarioLogado` por sessão (`UsuarioLogadoPerSessionLifeStyle`): se não há `HttpContext`/`Session`, cai num **`MemoryCache` do processo compartilhado** por todos; fora do JobHost isso mistura usuários. Em código novo, não resolva `IUsuarioLogado` em thread sem contexto.
- Controllers fazem `new BLLxxx()` (service locator por trás); alguns sem `using` -> contexto EF vivo até o fim da request (`PerWebRequest`). `DbContext` é por request: não guarde BLL em campo estático.
- `HomeController.Index` e outros testam `idEmpresa == 0 || usuarioContainer == null` na ordem errada (desreferencia antes do null). `LoginController` usa `login.Grupos.First()`: usuário sem grupo gera exceção não tratada (só `BusinessException` é capturada).
- Sessão InProc (sem `sessionState` no Web.config do MVC): reciclar o pool desloga todos (não verificado em produção).

## 4. Dados e SQL

- Muito SQL concatenado/inline em DAL (Dapper) e em controllers (`EmpresasController` monta connection string do tenant e do MyBI por `string.Replace`/interpolação). Prefira parâmetros.
- `WITH (NOLOCK)` generalizado nas leituras: leituras sujas em listagens.
- Fluxos que misturam central + tenant + AWS **sem transação** (criar empresa, transformar trial, publicar aviso): falha no meio deixa estado parcial. `PublicarAvisoCrm` engole exceção por empresa (`catch {}`); `Excluir` empresa é só `ativo=false` e devolve mensagem genérica no erro.
- Banco renomeado ao transformar trial (`smark_<subdominio>_<id>`): dependências de `dataBaseName` em jobs/conexões abertas.
- Scripts SQL: nomes fora do padrão, par ROLLBACK faltando, `GO` em alguns arquivos apesar da regra, nome de arquivo enganoso (`PadronizacaoPlural` faz singular). Esquecer o registro no csproj do WebForms = script fora do pacote. Ver `banco-de-dados.md`.
- Runner de migrations ordena por string e roda cada arquivo de uma vez; numeração >99 quebra a ordem lexical.
- EF automático desabilitado: nenhuma coluna nova aparece sozinha; entidade/Map/DbSet/script precisam andar juntos.

## 5. Código e manutenção

- Áreas/Controllers duplicados: `Permissoes` x `GerenciarUsuarios` (mesma lógica copiada), `Cadastros/*` x `Ferramentas/*`, `Empresas`/`Trials` ambos com "Transformar". Corrigir bug em um e esquecer o gêmeo é recorrente.
- `EmpresasController` (≈1,6 mil linhas) e `LoginsController` da Api (≈1,3 mil) concentram regra de negócio no controller; regra de MyBI é mista (controller + BLL).
- Retornos JSON sem contrato único (`sucesso`/`success`, `message`/`mensagem`/`error`): ler o JS da view antes de mudar.
- URL de API externa de trial hardcoded em `TrialsController.CriarTrial`; vários caminhos/mensagens fixos em português no código.
- Código comentado extenso (menus no `_Layout`, `CustomAuthorize`, trechos de Mongo/EF Audit). `ModuloMap`, `Helps`, `Suportes`, `Versoes` e telas WebForms indicam legado possivelmente sem uso (não verificado).
- `Web.config` do MVC com `compilation debug="true"`; `httpRuntime` declarado duas vezes; connection strings comentadas e dependência total do Parameter Store (sem AWS o app nem sobe).
- `node_modules` versionado (3,2 mil dos 3,6 mil arquivos do projeto MVC): diffs enormes ao atualizar Gentelella/TinyMCE; `bundleconfig.json` gera `.min` dentro de `node_modules`.
- Projetos fantasmas: `UI.Core`, `MVC.UI.Core`, `Api.Core`, `Tests/BackOfficeServicesTests` e `Tests/TestAppConsole` só têm `bin/obj`.
- **Sem testes** do BackOffice: `src/Tests/SiteExpress.SMark.Tests` referencia apenas Global.Common e a BLL do CRM; nenhuma classe de teste cobre BLL/DAL/controllers do BackOffice. Validação é manual (ver `roteiros.md`). A CI só faz build + VSTest (`trigger: none`).
- Dupla compilação: BLL/DAL/Domain/Infra/Ioc/ServicesProxy também compilam em `net10.0`; não vi na CI garantia de que o alvo net10 continua verde (não verificado).
- O WebForms legado e o WCF (`BackOfficeServices`) seguem no repo sem ficar claro se rodam em produção (não verificado).
