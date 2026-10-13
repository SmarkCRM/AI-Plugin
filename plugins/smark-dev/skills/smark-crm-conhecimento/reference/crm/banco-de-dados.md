> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Banco de dados: tenant (EMPRESAS) x central (BACKOFFICE)

Objetivo: explicar onde cada dado mora, como a conexão do tenant é montada, como a DAL acessa o banco e como os scripts de schema são organizados e (não) executados.

## Dois tipos de banco
| | EMPRESAS (um banco por cliente/tenant) | BACKOFFICE (central) |
|---|---|---|
| Conteúdo | Oportunidades, Contas, Contatos, Propostas, Funil, Tarefas, Templates, `Configuracoes`, `Logins` (do CRM), genéricos, importações... | `Empresas`, `Servidores`, `Menus`, `Permissoes`, `PermissoesEspeciais`, `Grupos`, logins do BackOffice, licenças, `QueueMessages`, `QueueProcess`, `QueuePrioridade`, `MigracaoBDBackOffice`/`MigracaoBDEmpresa` |
| Acesso no CRM | `SMarkContext` (EF6) e Dapper com `Contexto.ConnectionString` | via BackOffice.Api (login, grupos, licenças, empresas) e contexto `SMarkBackOfficeContext` na BackOffice.BLL/DAL/Infra |
| Quem escolhe a conexão | `DbConfiguration` por sessão | Parameter Store (`GetConnectString()` -> aplicação `APIBO`, base `SMarkBackOfficeContext`) com fallback para connection string do config; `DAL/ADO/BaseBancoDados` usa `ConnectionStrings["SMarkBackOffice"]` |
- Não existe `idEmpresa` nas tabelas do tenant: o isolamento é o próprio banco. Nunca abrir conexão com string fixa em DAL de CRM.
- No login, os dados do banco do tenant vêm da `Empresa` (campos `dataBaseName`, `dbServer`/`idServidorDataBase`, usuário e senha do banco). `LoginController` troca o nome do servidor por `ConfigurationParametersApplicationSingleton.GetServidor(idServidor)` (lista `servidoresDeBanco` do Parameter Store) e monta `DbConfiguration` (`DbName/DbServer/DbUser/DbPassword/DbProvider`). `IsValid` exige todos preenchidos.
- `DbConfiguration.GetConnectionString()` monta a string com `Encrypt=False`, `MultipleActiveResultSets=True`, pool e `Application Name=SMark.JobHost` **fixo** (mesmo na UI). Contém credencial interpolada em memória, nada versionado ali.
- `SMarkContext`: `AutoDetectChangesEnabled=false`, `ValidateOnSaveEnabled=false`, `CommandTimeout` = chave `JobHostCommandTimeout` (padrão 300 s), initializer `CreateDatabaseIfNotExists`, convenção `DateConvention` (DateTime -> `datetime`), remove `OneToManyCascadeDeleteConvention`, `Configurations.AddFromAssembly` (todo `*Map` do assembly é carregado). 162 `IDbSet<>` no `SMarkContext` e 157 em `ISMarkContext`: `IDbSet` só é necessário se alguma DAL acessar `Contexto.<Set>` direto (nesse caso, adicionar nos dois); entidades novas como `Template` não têm `IDbSet` e funcionam via `Contexto.Set<T>()` do `DALBase<T>`.

## Acesso a dados na DAL (três estilos, todos em uso)
1. EF6 LINQ via `Contexto` (CRUD genérico em `DALBase<T>`: `Add`, `Update` marca `Modified`, `Delete` é **hard delete**).
2. Dapper 2.0.123 sobre `new SqlConnection(Contexto.ConnectionString)` — pesquisas pesadas (cards, kanban, relatórios, gráficos). `DALOportunidades` tem 8.492 linhas.
3. ADO.NET puro/`Database.SqlQuery` (relatórios com `DataTable`).
- SQL em resources: `DAL/Resources/Querys.resx`, `QuerysOportunidade.resx` (placeholders `?Parametros?`, `?ParametroFilho?`, `?ParametroOrderBy?`).
- Versões: EntityFramework 6.5.1, Z.EntityFramework.Plus.EF6 9.103.9.1.
- `WITH (NOLOCK)`: ~360 ocorrências nos `.cs` da DAL (210 só em `DALOportunidades`).
- Pesquisa de oportunidades: `montarfiltro` e `MontarFiltroFilho` (privados em `DALOportunidades`) alimentam `Pesquisar`, `PesquisarKanban`, `PesquisarKanbanPorEtapaFunil`, `PesquisarCount`, `PesquisarTotalPropostas` e `GetSqlPesquisarCount`. Filtro novo vai ali. Datas entram como `CONVERT(DATETIME,'{0:yyyy-MM-dd 00:00:00}',120)` com `DateTime?` tipado; paginação por CTE + `OFFSET/FETCH`.
- **Risco verificado**: o texto de busca livre (`palavra`, `_.Trim()`) é interpolado dentro de `LIKE '%...%'` em `montarfiltro` (só aspas simples são removidas em parte dos ramos). Tratar como SQL injection potencial; em código novo usar `@param`/`DynamicParameters`.

## Armadilhas do `DALBase<T>`/contexto (verificadas)
- `CountAsync(Expression)` ignora a expressão e conta tudo (`DALBase.cs` ~linha 464).
- `RetrievePaginate(Expression, pageSize, pageCount)` lança `NotImplementedException`; a sobrecarga com `orderby` funciona.
- `Dispose` não descarta o contexto (é do container): só zera a referência. Não colocar `using` no contexto injetado.
- Com `AutoDetectChangesEnabled=false`, alteração em entidade rastreada só persiste se chamar `Update`.
- `Menus`, `QueueMessages` etc. não estão no `SMarkContext` do CRM; (verificado: sem `IDbSet`) ficam no contexto do BackOffice.

## Entidades e mapeamentos
- `Domain/Models` (~200 classes) <-> `Infra/Mappings/*Map.cs` (175; `EntityTypeConfiguration<T>`, sempre `ToTable` + `HasColumnName`). Legado: colunas camelCase em português (`idOportunidade`, `dataVigencia`, `deleted`); tabelas novas: `Id<Tabela>`, `DataCriacao`, `IdLoginCriacao`, `Deleted` (ex.: `Templates`).
- Nomes de tabela: PascalCase plural (`Oportunidades`, `Contas`, `Logins`, `Configuracoes`, `Templates`).
- `Oportunidades` guarda oportunidade e pós-venda (`tipo`).
- Config por tenant: tabela `Configuracoes` (`nomeConfiguracao` = valor do enum `NomeConfiguracao`, `valor`, `tipoDado`, `pagina`, `idLogin`); leitura por `BLLConfiguracao.RetrieveByNome(NomeConfiguracao.X)`.

## Menus e permissões (BACKOFFICE)
- `Menus`: `idMenu`, `menu` (texto exibido), `Aplicacao`, `enumMenu` (valor do enum `Menus` em `Global/General/Enum/Menus.cs`, ~100 itens; ex.: `Templates = 102`), `idPai`, `path` (ex.: `~/BancoDeDados/Templates`), `ordenacao`.
- `Permissoes`: (`idGrupo`, `idMenu`, `tipo` = `TipoPermissao`: Adicionar 1, Editar 2, Visualizar 3, Deletar 4). `PermissoesEspeciais` ligam grupo a `TipoPermissaoEspecial` (enum em `Global.ServicesDTO/TipoPermissao.cs`). `Grupos` têm `admin`, `adminCrm`, `idHorarioAcesso`, `descontoMaximoPermitido`.
- Tela nova = linha em `Menus` com `path` contendo o nome do controller + permissões do grupo; senão `CustomAuthorize` nega. Texto do submenu = `Menus.menu`; ordem alfabética dos filhos em `Views/Shared/_Menu.cshtml` (`OrderBy(x => x.menu)`); menus ficam na sessão, então só atualizam após novo login.
- Script de menu: `INSERT INTO dbo.Menus (menu, Aplicacao, idPai, path, ordenacao, enumMenu) ...` com `IF NOT EXISTS` (padrão visto em `Scripts por Sprint/3.32.13`).

## Scripts de schema
Local: `src/BackOffice/UI/Sql/`
- `Scripts por Sprint/<versão>/` (prática atual: 3.33 é a mais recente, de out/2026; também pastas de patch como `3.32.13`, hotfix `3.32.4-hotfix`). Nome: `NN - (EMPRESAS|BACKOFFICE) Verbo-Objeto-MIGRATE.sql` e `...-ROLLBACK.sql`, registrados como `<Content>` no `src/BackOffice/UI/SiteExpress.SMark.BackOffice.UI.csproj` (~974 entradas). Nomes têm pequenas inconsistências (espaço antes de `.sql`, rollback com nome diferente do migrate em algumas pastas antigas); conferir antes de assumir par.
- Scripts recentes são idempotentes (`IF NOT EXISTS (... sys.columns ...)`), sem `GO` e usam `EXEC('...')` para DDL dentro de blocos `IF`. Scripts antigos (61 arquivos) contêm `GO`.
- `Migrations/{_Setup,BackOffice,Empresas}`: sistema antigo, sem alteração desde 2021 (último `Empresas` numerado 112; `BackOffice` até 22). Runner: `BackOffice/UI/Views/Migrations/RunMigrations.aspx.cs`.
- `ScriptsBase/`, `SmarkScriptBase.sql`, `SmarkCria*`, `BackOffice.sql` (UTF-16 LE: `grep` do bash não acha texto, usar `rg`/Grep): bases para criar empresa/trial.
- Regra do guia `doc/guides/MIGRATIONS.txt`: sempre par MIGRATE/ROLLBACK, numeração incremental, **sem `GO`** (o cliente .NET não entende).

### Quirks do runner `RunMigrations`
- Só lê `Sql/Migrations/*`; **não lê `Scripts por Sprint`**. Como os scripts de sprint são aplicados em produção não está no repositório (não verificado: provavelmente manual/SSMS ou pipeline externo).
- Controle por nome do último script em `MigracaoBDBackOffice`/`MigracaoBDEmpresa` (por empresa). `ListarArquivosAindaNaoExecutados` ordena **como texto** (descendente) e para no último executado; só executa arquivos com `MIGRATE` no nome. Numeração com 3 dígitos ou nomes fora do padrão quebra a ordem.
- Executa o arquivo inteiro com `ExecuteNonQuery`: `GO` falha. Valida que cada prefixo numérico aparece exatamente duas vezes (MIGRATE+ROLLBACK).
- Percorre as bases de todas as empresas (`Empresas` do BACKOFFICE) e aborta na primeira falha (`throw`), interrompendo as demais.

## Como escolher a pasta e o banco de um script novo
- Coluna/tabela de negócio do cliente -> `(EMPRESAS)` (roda em todas as bases; idempotente e leve; coluna nova `NULL` ou com default).
- Menus, permissões, empresas, fila -> `(BACKOFFICE)`.
- Pasta = sprint/patch do work item (campo `System.IterationPath` do Azure DevOps); se não existir, criar com numeração desde `01`; não escolher "a mais recente" por ordem alfabética. Sem work item, perguntar. Mudança de coluna lida por EF exige também `Models/X.cs` + `XMap.cs`.
- Depois de criar tabela/coluna: `Models/X.cs` + `XMap.cs` (e `IDbSet` só se a DAL usar `Contexto.<Set>`).
