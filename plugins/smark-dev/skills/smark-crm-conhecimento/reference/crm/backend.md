> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Backend: Controller -> BLL -> DAL, IoC e padrões de código

Objetivo: explicar como uma requisição atravessa as camadas, como o Castle Windsor monta as dependências, e os padrões de erro/validação/cópia de objetos usados no CRM legado.

## Fluxo de uma requisição
1. Rota de Area -> `WindsorControllerFactory` (`Web/App_Start`): procura no assembly da UI um tipo chamado `"I" + NomeDoController` e, se estiver registrado no Windsor, resolve por ele (injeção por construtor). Senão usa o construtor padrão do MVC. São 67 controllers registrados em `RegisterTypesConfig.cs` (212 linhas `new RegisterType(...)`, 137 delas de `IBLL*`).
2. Filtros globais (`App_Start/FilterConfig.cs`), nesta ordem: `HandleErrorAttribute`, `ActionLogFilter` (classe vazia, corpo comentado), `CustomAuthorize`, `CountFilter` (só com `IsCountFilter=true`), `AcessoUnicoFilter` (só com `IsAcessoUnicoFilter=true`), `VerificarSenhaExpiradaAttribute`; mais filtros CSP se `IsCspFilter=true`. `HorarioDeAcessoFilter` existe mas **não está registrado nem usado como atributo**.
3. `BaseController.BeginExecute` força a cultura `pt-BR` e chama `InicializarPermissoes`, **engolindo qualquer exceção** (se falhar, `ViewBag.Adicionar/Editar/Deletar` ficam sem valor, sem erro visível).
4. Action monta/valida a VM, converte strings de tela (ex.: período `dd/MM/yyyy - dd/MM/yyyy`) e chama a BLL.
5. BLL (`BLLBase<T>`) valida e usa o repositório (`IDAL<X>`) ou converte VM -> DTO (`Global.ServicesDTO.*DTO`) para SQL Dapper.
6. DAL (`SmarkDALBase<T>` : `DALBase<T>`) usa `ISMarkContext` (EF6) ou `new SqlConnection(Contexto.ConnectionString)` com Dapper.
7. Retorno: `View`/`PartialView`/`Json` ou `MensagemVM.Json(...)`.

## IoC (Castle Windsor) — `SiteExpress.SMark.Ioc/WindsorFactory.cs`
- Singleton `WindsorFactory.Instancia`; `Iniciar()` roda em `Application_Start` depois do Parameter Store. `RegisterTypesConfig.Init()` registra BLLs/controllers/wrappers explícitos.
- **Dois containers**: `Container` quando `HttpContext.Current != null` (lifestyle `PerWebRequest`; em net10 `Scoped`) e `ContainerThread` fora do contexto web (lifestyle `PerThread`). Código que roda em `Task.Run`/Hangfire/JobHost cai no segundo.
- DALs são registradas por convenção (`Types.FromAssemblyContaining(typeof(SmarkDALBase<>)).BasedOn(typeof(IDALBase<>)).WithServiceAllInterfaces()`); `ISMarkContext` -> `SMarkContext` por request; `IDbConfiguration` -> `DbConfiguration` com lifestyle `SmarkContextDBConfPerSessionLifeStyle` (guardado na **Session**, chave com prefixo `SmarkContextDBConfPerSessionLifeStyle`; fora do web, em `MemoryCache`); `IUsuarioLogadoSession` -> `LoginSessionDTO` por sessão (`UsuarioLogadoPerSessionLifeStyle`).
- `Resolver<T>()` devolve `null` (não lança) se o container não foi iniciado; erros viram `Exception("WindsorFactory - T Resolver<T>()")`.
- Também existe `App_Start/ContainerBootstrapper` + `WindsorActivator` (`PreApplicationStart`, `Install(FromAssembly.This())`): bootstrap Windsor separado do `WindsorFactory`, sem papel claro (não verificado).
- `SiteExpress.SMark.Ioc.Unity` não é usado.

## Dois estilos de BLL (escolher pelo vizinho)
- Antigo: BLL injetada por interface (`IBLLOportunidades<Oportunidade>` -> `BLLOportunidades`) e registrada em `RegisterTypesConfig`. Muitos controllers herdam `BaseController(...)` com 7 dependências injetadas, ou o ctor vazio que resolve tudo por `WindsorFactory.Resolver`.
- Novo: BLL com ctor sem parâmetros: `this.Repositorio = WindsorFactory.Resolver<IDALX>()` e uso por `using (var bll = new BLLX())` (exemplo: `BLLTemplates`). Segundo ctor `(ISMarkContext contexto, bool isDispose)` permite reutilizar o contexto (JobHost, threads, filtros).
- `BLLBase<T>` (`BLL/BLLBase.cs`): CRUD (`Add/Update/Delete/Retrieve/RetrieveSimple` + versões `Async`), `Validar(entidade, Operacao)` -> abstratos `ValidarAdd/ValidarUpdate/ValidarDelete`. Atenção: `Update(entidade, bool validar = false)` **não valida por padrão** (`BLLOportunidades` sobrescreve com `validar = true`); `AddSemValidacao/UpdateSemValidacao` pulam validação. `BLLBase` também carrega lógica de e-mail de notificação (`EnviarEmailNotificacao`, `EnviarEmailTransferenciaDeCarteira`, montagem de e-mails) e envia por Hangfire só se `IsHangfire=true`, senão sincronamente via `BLLEmails`.
- `Dispose()` zera `Repositorio`; sempre usar `using` ao instanciar com `new`.

## Camada de dados (resumo; detalhe em `banco-de-dados.md`)
- Entidade nova: `Domain/Models/X.cs`, `Domain/Interfaces/Repositorios/<Mod>/IDALX.cs` (`: IDALBase<X>`), `Infra/Mappings/XMap.cs`, `DAL/<Mod>/DALX.cs` (`: SmarkDALBase<X>, IDALBase<X>, IDALX`, dois ctors `(ISMarkContext)` e `(ISMarkContext, bool)`), `BLL/Interfaces/<Mod>/IBLLX.cs` + `BLL/<Mod>/BLLX.cs`. DAL e Map entram por convenção (sem registro).
- Libs são SDK-style: arquivos novos entram sozinhos. Só a UI exige `Compile Include`.

## Validação e erros
- `BusinessException` (`Global/General/Exceptions`): `new BusinessException("Msg_Chave")` (padrão: tipo Error), com `MensagemTipo`, ou `(msg, ModelErrors)` (vira Warning). Há 408 `throw new BusinessException` na BLL; os mais comuns: `Msg_ValorDuplicado` (63), `Msg_Validacao` com `ModelErrors` (33), `Msg_ExistemDependencias` (34), `Msg_CamposObrigatorios` (14).
- A BLL lança a **chave**; a UI resolve o texto: `MensagemVM.Json(ex)` chama `UIHelper.ObterResourceMensagens(chave)` em `Web/App_GlobalResources/Mensagens.resx`; se a chave não existir, **mostra a própria chave** (sem erro). Chave nova usada na BLL precisa existir no `Mensagens.resx` da Web (existe também `Global/General/Resources/ResourceMessage.resx`, usado por `ResourceHelper`/APIs). Com `isKeyMessage: true` a string é tratada como texto pronto (não busca resource).
- `MensagemTipo`: `Nenhum 0, Sucess 1, Warning 2, Error 3, Info 4` (grafia `Sucess`). `MensagemVM.Json` monta `{ sucesso, script: "seBase.mostrarMensagem('#divMsg<Tipo>','...')", mensagem, tipo, id, callBack, mensagemCallBack }` (aspas simples escapadas, quebras de linha trocadas por espaço). `MensagemVM` é `internal` e tem overloads que lançam `NotImplementedException` (`Json(MensagemTipo, object)`).
- Erro em formulário: `BaseController.ViewModelError<T>(formView, ex, entity)` devolve a partial com erros.
- Cópia de objetos: não há AutoMapper. `CopiaPorValor.CopiarValores<TDest, TOrig>` (copia por nome e **ignora coleções/navegações**) e `Omu.ValueInjecter` (`InjectFrom`) no login e na BackOffice.Api. Listas e objetos aninhados são copiados à mão.
- Filtro de pesquisa: `*SearchVM` (BLL/Models) espelha `*SearchVMDTO` (Global.ServicesDTO); a BLL copia campo a campo em **cada** método de pesquisa. Filtro novo precisa passar por todos (lista, kanban, kanban por etapa, count, total de propostas).
- Entidades novas usam soft delete `deleted` e auditoria `DataCriacao/IdLoginCriacao/DataAlteracao/IdLoginAlteracao` (padrão visto em `Template`); `BLLBase.Delete(id)` valida (`ValidarDelete`) e chama `DALBase.Delete`, que faz **exclusão física** (`EntityState.Deleted` + `SaveChanges`). Soft delete só acontece onde a BLL/DAL específica marca `deleted` (`MarkAsDeleted`, `UPDATE ... SET deleted = 1`).

## Base de controllers (`Web/Controllers/BaseController.cs`, 3.083 linhas)
Sessão (`ISessionWrapper`: `UsuarioLogado`, `UsuarioLogadoInstancia`, `Menus`, `MenusCards`), permissões (`isAdmin`, `isAdminCRM`, `isAdminOrAdminCRM`, `supervisionados` — **lidos de cookies**, ver `seguranca-sessao-config.md`; `VerificarTipoPermissaoMenu(Menus, TipoPermissao)` usa `MenusCards`), listas de combos (`ListarItensDeLogin/Canal/Enumerador/ProdutoServico`), campos personalizados (`ParseGenericosCamposValores`, `AddOrUpdateGenericosValores`), tabelas dinâmicas (actions públicas), tags, integração de e-mail (Gmail/Office365), tokens para APIs Mark/Queues (`AutenticarAPIMark`, `AutenticarAPIQueues`), log de auditoria (`LogSystemAsync`), `Menu()` (child action que monta o menu; injeta "Relatórios/Acompanhamento de Relatórios" para quem tem acesso ao Assistente).

## Padrões de código a copiar
- Controller de cadastro de referência: `Areas/BancoDeDados/Controllers/TemplatesController.cs` (`[CustomAuthorize]` na classe e `AccessLevel` por action; `List` retorna jqGrid via `MvcJqGrid.GridSettings`).
- Painel com filtros: `Areas/Comercial/Controllers/OportunidadesCardsController.cs` (conversão do período em método privado de tratamento de datas, chamado por todas as actions de pesquisa).
- Constantes de grafia a respeitar: `MensagemTipo.Sucess`, `#divMsgSucess`, `Conta.carteiraExcusiva` (grafia do banco).
- Código novo em BLL/DAL/Domain/Infra deve compilar em net48 e net10.0.
- Hangfire: `IsHangfire=false` no `Web.config` versionado; quando true, `Global.asax` cria o banco `DbHangfire` no servidor e sobe `BackgroundJobServer`.
- Logs: log4net (`XmlConfigurator.Configure()` no start), Serilog -> Seq (`Util/LogService`, chaves `serilog:*`), auditoria em `LogSystem` (`IsLogSystem`). Mongo: contextos existem em `Infra`, mas todos os consumidores estão comentados; não reativar.
