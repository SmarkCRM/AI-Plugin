# BackOffice 4.8 — bancos, tabelas e scripts SQL
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: dizer quais bancos o BackOffice usa, quais tabelas centrais ele administra e como os scripts SQL (por sprint e migrations) são escritos, registrados e executados.

## 1. Bancos

| Banco | Como é acessado | Observação |
|---|---|---|
| **BACKOFFICE (central)** | EF6 `SMarkBackOfficeContext` + Dapper/ADO com `ConfigurationParametersApplicationSingleton.GetConnectString()` (aplicação `APIBO`, nome `SMarkBackOfficeContext`) | Único banco do dia a dia do BackOffice/BackOffice.Api. A connection string vem do **Parameter Store** quando `UsarParameterStore=true`; senão, `ConnectionStrings["SMarkBackOfficeContext"]` do config (no MVC está comentada). Valores: nunca registrar. |
| **Bancos das empresas (tenants)** | `SqlConnection` montada na hora a partir de `Empresas` (`dataBaseName`, `dataBaseUser`, `dataBasePassord`) + `Servidores.nomeServidor` | Um banco por empresa. O BackOffice escreve neles em: publicar aviso (`BLLAviso.PublicarAvisoCrm` -> tabela `Notificacoes` do tenant, com `catch` vazio), transformar trial (renomear banco), criar usuário de banco para o MyBI, criar empresa (rodar scripts base). Tabelas `LoginContador`, `Notificacoes`, `Telefones`, `QueueFiles` pertencem ao tenant. |
| Log (`SMarkLog.edmx`) | `Domain/SMarkLog*`, `Infra/SMarkLogContexto` | Contexto EF de log legado; uso atual: (não verificado). |
| MongoDB | appSettings `MongodbConnectionString`, `SMarkMongoDbDatabaseName` ainda presentes nos configs; `Domain/MongoModels/GrupoEntity.cs` | Sem uso ativo confirmado no BackOffice (não verificado). |

Hosts diferentes (MVC, Api, JobHost) apontam, via Parameter Store, para o **mesmo** BACKOFFICE do ambiente. Mudar schema/dados aqui afeta CRM, Api e JobHost ao mesmo tempo.

## 2. Tabelas centrais do BACKOFFICE (mapeadas em `Infra/Mapping/*Map.cs`)

| Grupo | Tabelas | Notas |
|---|---|---|
| Empresas/tenants | `Empresas`, `Servidores`, `CadastrosTrial`, `Licenciamentos` | `Empresas`: `subdominio`, `ativo`, `trial`, `isDemo`, `PlanoCliente` (0 Premium, 1 Enterprise, 2 Connection), `trialAvailable`/`trialOrder` (pool de empresas trial pré-criadas pelo JobHost), `dataBase*`, `idServidorDataBase`/`idServidorAplicacao`, licenças CRM e BI (`licencas`, `numLicencasInicial`, `licencasBI`, `numMinimoLicencasCrm`), `limiteTokensAI`, `DiasExpiracaoSenha`, `NotificarExpiracaoDias`, `DataInativou`, `DataVirouEmpresa`, `dbOrigem`. `Licenciamentos` = solicitações de aumento/redução de licença. |
| Acesso | `Logins`, `Grupos`, `GruposLogin`, `Menus`, `MenusGrupo`, `Permissoes`, `PermissoesEspeciais`, `GruposPermissoesEspeciais`, `Modulos`, `LoginSessions`, `LoginHistoricoAcessos`, `LoginHistoricoSenhas`, `LoginTentativaAcessos` | **`Logins` guarda usuários do CRM (`Aplicacao=1`) e do BackOffice (`Aplicacao=2`)**; enum `TipoAplicacao`: 1 Smark, 2 Backoffice, 3 Service, 4 API. `Logins.replicar` marca usuário a ser copiado para o banco do tenant. Campos de política de senha (bloqueio, expiração, forçar troca) foram adicionados na sprint 3.32. |
| Menus/permissões | `Menus` (`idMenu`, `menu`, `Aplicacao`, `idPai`, `path`, `ordenacao`, `enumMenu`), `Permissoes` (`idGrupo`, `idMenu`, `tipo`: 1 adicionar, 2 editar, 3 visualizar, 4 deletar) | O texto do menu exibido no CRM é a coluna `menu` (dado, não resource). O CRM carrega menus no login por `MenusGrupo`/`Menus`/`Permissoes` (`DALLogin.RealizarLogin`). O enum `Menus` está em `Global/General/Enum/Menus.cs`. |
| Módulos contratados | `ModulosSistema` (`enumModulo`, `nome`), `ModulosSistemaEmpresas` | `enumModulo` espelha o enum `ModulosDoSistema`, **definido no CRM** (`SiteExpress.SMark.Domain/Enumeradores/Enumeradores.cs`: 1 PainelDeContas, 2 PainelDeOportunidade, 3 PainelDePosVenda, 4 DeclaracaoDeSaude, 5 SimulacaoDePlanoDeSaude, 6 AssistenteIA, 7 DashboardSmarkBI). |
| Fila do JobHost | `QueueProcess` (tipo -> classe), `QueueMessages`, `QueuePrioridades`, `JobHostHeartbeat` | Ver `backoffice-api-e-servicos.md`. `QueueMessages` ganhou colunas de controle (lock, tentativas, próxima tentativa) na 3.32. |
| Dicionário de campos | `SmarkEntidades`, `SmarkCampos`, `SmarkEntidadesRelacionamentos`, `SmarkObrigatorios`, `SmarkEntidadesPais` | Metadados do CRM servidos via Api `Campos`. |
| Conteúdo/suporte | `Conteudos`, `Culturas`, `Helps`, `Imagens`, `Envios`, `Suportes`, `IteracoesSuporte`, `TarefasSuporte`, `Versoes`, `ScriptsDB`, `Configuracoes` | Em boa parte legado WebForms. |
| Engajamento/uso | `UsoFuncionalidade`, `RelatorioEngajamento`, `Funcionalidade`, `GrupoFuncionalidade`, `Onboards`, `OnboardsEmpresas` | |
| Notificações/avisos | `NotificacoesAgendadas`, `Avisos`, `AvisoEmail`, `AvisoEmailEnvioLog`, `HistoricoEmailsDoSistema`, `HistoricoEnvioEmailsTrial`, `BlackListEmails` | `Avisos*` são acessadas por SQL direto (sem `DbSet`). |
| BI | `UsuariosSmarkBI`, `AbasMyBI` | Usuário/credencial do MyBI por empresa (a coluna de senha é texto, não hash). |
| IA/auditoria | `LogTokensAI`, `AuditoriaLimiteTokensAI`, `AuditoriaConsultaCNPJ`, `Cnae`, `CIDs`, `MunicipiosIBGE`, `EstadosIBGE` | |
| Migrations | `MigracaoBDBackOffice`, `MigracaoBDEmpresa` | Metadados do runner (ver §4). |

## 3. Scripts por Sprint (prática atual) — `src/BackOffice/UI/Sql/Scripts por Sprint/<versão>/`

Verificado nos arquivos (91 pastas de `2.89` até `3.33`; ~929 `.sql`):

- **Pasta** = versão/sprint (`3.33`, `3.32.13`, `3.32.4-hotfix`...). Criar a pasta se o patch ainda não tem uma.
- **Nome** (≈80% dos arquivos): `NN - (EMPRESAS) Nome-Descritivo-MIGRATE.sql` e `NN - (EMPRESAS) Nome-Descritivo-ROLLBACK.sql`; o alvo entre parênteses é **`(EMPRESAS)`** (roda em todos os bancos de clientes) ou **`(BACKOFFICE)`** (banco central). NN sequencial dentro da pasta.
- **Quirks reais (não copie)**: sufixo com ponto (`...Tabela.MIGRATE.sql`), espaço antes do `.sql` (`MIGRATE .sql`), typos (`(EMRPESAS)`, `(BACKOFICCE)`), `NN -Nome`, arquivo `...-MIGRATIONS.sql`, par ROLLBACK ausente (500 MIGRATE x 418 ROLLBACK), subpasta `3.32/BACKOFFICE`. O nome do `3.33/03 ... Update-Menus-BancoDeDados-PadronizacaoPlural` na verdade renomeia para o **singular**.
- **Conteúdo**: T-SQL idempotente (`IF NOT EXISTS (SELECT 1 FROM sys.columns ...)` / `IF EXISTS (SELECT 1 FROM Menus WHERE ...)`); UTF-8 com BOM e CRLF nos recentes. O `doc/guides/MIGRATIONS.txt` manda **não usar `GO`**; 61 scripts do repo ainda têm `GO` (inclusive `3.32/10 ... JobHostHeartbeat`), então confirme antes de assumir.
- **Registro no csproj**: cada arquivo é listado em `src/BackOffice/UI/SiteExpress.SMark.BackOffice.UI.csproj` como `<Content Include="Sql\Scripts por Sprint\<pasta>\NN - %28BACKOFFICE%29 Nome-MIGRATE.sql" />` (parênteses codificados `%28`/`%29`). Verifiquei: 1.434 dos 1.436 arquivos de `Sql/` estão registrados; só 2 scripts antigos de `3.2/10` não. Esquecer o registro = script não vai no pacote de deploy do WebForms.
- **Quem executa**: nenhum código do repo lê `Scripts por Sprint`; a aplicação deles em HLG/PROD é manual/processo externo (não verificado quem e quando). Por isso: o script tem de rodar sozinho em qualquer banco do alvo, ser idempotente e rápido.
- `Sql/BackOffice.sql` (UTF-16 LE, ~330 KB) é dump de referência do schema/dados do BACKOFFICE; `Sql/ScriptsBase/00..05-*.sql` (+ `SmarkScriptBase.sql`) criam banco de empresa nova/trial; `Sql/Clientes` guarda scripts pontuais de cliente. Se atualizar o dump, preserve a codificação.
- Alterou coluna lida pelo EF? Atualize também `Domain/Models/X.cs` + `Infra/Mapping/XMap.cs` (e DTOs/SQL de Dapper). Alterou tabela do **tenant**? O CRM (`src/SMark/...DAL/Infra`) também precisa conhecer.

## 4. Migrations automáticas (mecanismo antigo) — `Sql/Migrations`

- Runner: `src/BackOffice/UI/Views/Migrations/RunMigrations.aspx.cs` (WebForms), chamado pelo build/deploy com um parâmetro fixo na querystring (segredo no código — só o caminho importa; ver `seguranca-e-acesso.md`).
- Passos: cria `MigracaoBDBackOffice`/`MigracaoBDEmpresa` se faltarem (`_Setup`); exige que cada número tenha exatamente 2 arquivos (MIGRATE+ROLLBACK) em `Sql/Migrations`; roda os `*MIGRATE*` da pasta `BackOffice` mais novos que o último registrado; depois percorre todas as empresas (`Empresas` join `Servidores`) e roda os da pasta `Empresas` em cada banco que existir.
- Ordenação **por nome (string)** decrescente até achar o último executado; cada arquivo é lido inteiro e executado de uma vez (sem separar `GO`). Estado: `Migrations/BackOffice` termina em `22-...` e `Migrations/Empresas` tem 221 arquivos; a prática recente é `Scripts por Sprint`. Se o runner ainda é usado no deploy: (não verificado). Não adicione scripts novos em `Migrations` sem confirmar com a equipe.

## 5. Criação de empresa/trial (onde os scripts base entram)

- `BLLEmpresas.Add(...)` cria grupos padrão ("Administrador CRM", "Administrador", "Vendedores"), cria CNAME no Route 53, roda scripts base (`pathSql`) para criar o banco do tenant e copia permissões da empresa-modelo (appSetting `empresaImportacao`).
- Trials: o JobHost (`AddEmpresaTrialProcess`) mantém um pool de empresas trial pré-criadas (`trialAvailable=1`, limite `MaxEmpresasTrials`); `BLLEmpresas.CriaEmpresaTrial` (chamado pela Api `Empresas/CreateTrial`) reaproveita uma, renomeia subdomínio/login e gera senha. `MVC/SQL/Trial/Limpar _BASE_TRIAL.sql` limpa dados do banco-base.
