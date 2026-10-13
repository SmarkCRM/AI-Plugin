# BackOffice 4.8 — roteiros (checklists de investigação e planejamento)
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: passos práticos para investigar bug, planejar melhoria, quais perguntas fazer antes e quais riscos checar por camada, incluindo impacto no CRM. Consulte os demais arquivos da pasta para detalhes.

## 1. Investigar um bug no BackOffice

1. **Classifique a origem**: tela do BackOffice MVC? chamada à BackOffice.Api feita pelo CRM? item de fila do JobHost? script SQL? Cada um tem um ponto de entrada diferente (§5).
2. **Localize o ponto de entrada**: URL `/<Area>/<Controller>/<Action>` -> `MVC/Areas/<Area>/Controllers/<Controller>Controller.cs`; chamada de API -> `api/v1/<Controller>/<Action>` em `src/Services/SiteExpress.SMark.BackOffice.Api/v1/Controllers`; fila -> `QueueProcess.classe` em `SiteExpress.SMark.Services/Process/QueueProcess`.
3. **Desça a cadeia**: controller -> `BLLxxx` (`BLL/<Area>`) -> `IDALxxx` (`Domain/Interfaces/Repositorios`) -> `DALxxx` (`DAL/DAL/<Area>`) -> SQL (inline, `DAL/Resources/*.resx`, `DAL/SQLs`) ou EF (`Infra/Mapping`).
4. **Confirme qual banco**: central (`GetConnectString`) ou tenant (connection montada de `Empresas`+`Servidores`). Bug "só em um cliente" costuma ser dado/schema do banco dele (script `(EMPRESAS)` não aplicado).
5. **Verifique dados reais** (somente leitura): `Logins`, `Empresas`, `Menus`/`MenusGrupo`/`Permissoes`, `QueueMessages` (colunas `status`, `messageErro`, `lockedBy`, `tentativas`), `LoginTentativaAcessos`, `LoginHistoricoSenhas`.
6. **Reproduza**: suba o MVC (49981) e, se o fluxo envolve o CRM, o CRM (62662) + BackOffice.Api (1999) com o Parameter Store do seu ambiente. Para login, veja `ValidaLogin` (bloqueio por 5 falhas, expiração, licenciado) antes de supor bug de credencial.
7. **Cheque o gêmeo**: o mesmo código existe em `Permissoes` x `GerenciarUsuarios`, `Empresas` x `Trials`, MVC x WebForms? Corrija todos os que se aplicam.
8. **Considere sessão/cache**: menus ficam em `localStorage` (MVC) e na sessão do CRM; usuário precisa sair/entrar para ver dado alterado.
9. Logs: `LogHelper.LogMessage` (login/empresa), log do JobHost (Serilog em `LogService`), Elmah/Application Insights (não verificado destino em produção).

Sintomas comuns -> onde olhar:
- "Menu não aparece/aparece errado" -> `Menus`/`MenusGrupo` (idPai = `BackOfficeIdMenuPai` no BackOffice), `LoginController.LogarUsuario`, `_Layout.cshtml`; no CRM `MenuVM`.
- "Usuário não consegue logar" -> `BLLLogin.ValidaLogin`/`ControleBloqueioConta`/`ControleExpiracaoSenha`; `licenciado`, `ativo`, `deleted`, grupos; Api: `AutenticarUsuario.Validate`.
- "Exportação/importação travada" -> `Ferramentas/ListaJobhosts`, `QueueMessages.lockedBy/tentativas/proximaTentativa`, serviço `JobHostServices`, `JobHostHeartbeat`.
- "Empresa trial não criada" -> `trialAvailable`, `AddEmpresaTrialProcess`, `MaxEmpresasTrials`, Route 53 (`Route53Service`).
- "Usuário novo do BackOffice não chegou ao CRM" -> flag `Logins.replicar` e `ReplicarLogins` do CRM.

## 2. Planejar uma melhoria

1. Defina **quem usa** (operador do BackOffice, cliente via CRM, JobHost) e **onde a mudança deve aparecer** (tela, API, fila, relatório).
2. Liste as camadas tocadas: view/JS -> controller/VM -> BLL -> interface IDAL -> DAL/SQL -> entidade/Map/DbSet -> script SQL -> (se API) DTO + `*Service` do CRM.
3. Decida **schema**: coluna/tabela nova? Script por sprint com par MIGRATE/ROLLBACK idempotente e alvo `(BACKOFFICE)` ou `(EMPRESAS)`; registrar no csproj do WebForms; atualizar entidade + Map; avaliar o dump `Sql/BackOffice.sql`.
4. Decida **permissão**: tela nova = controller + view + linha em `Menus` (script BACKOFFICE) + vínculo a grupos (`MenusGrupo`) + decidir se fará checagem de acesso no `Index` e nas actions JSON (hoje não é padrão — proponha explicitamente).
5. Se for dado financeiro/licença/credencial: pensar em auditoria e em transação (os fluxos atuais não são transacionais com AWS/tenant).
6. Defina **como validar** (não há testes automatizados do BackOffice): roteiro manual por tela + SQL de conferência antes/depois + checagem do impacto no CRM.
7. Plano de **deploy**: ordem entre script SQL (BACKOFFICE/EMPRESAS), BackOffice.Api, MVC, JobHost, CRM; compatibilidade durante a janela (API antiga x CRM novo).

## 3. Perguntas a fazer antes de planejar

- Qual ambiente/cliente e qual versão (pasta de sprint) está em jogo? O script entra em qual pasta `Scripts por Sprint`?
- O dado é do banco **central** ou do **tenant**? Vale para todos os clientes ou só alguns (plano, módulo, trial)?
- Quem executa os scripts SQL em HLG/PROD e quando (o repo não mostra)? Há janela de deploy?
- A mudança afeta o CRM (menus, licenças, login, campos, notificações)? Alguém do CRM valida?
- O WebForms legado ou o WCF ainda são usados em produção para este recurso? (não verificado no código)
- Precisa de permissão nova? Qual grupo (`admin`, outros) deve ver? Isso já bloqueia no servidor ou só esconde no menu?
- Há integração externa envolvida (Route 53, S3, MyBI, Zimbra, API externa de filas, SMTP)? Existe ambiente de teste dela?
- Envolve senha/credencial? Seguir política (12+ caracteres, histórico 5, bloqueio 5, expiração) e não registrar valores.
- A tela/fluxo existe em duplicado (`Permissoes`/`GerenciarUsuarios`, `Empresas`/`Trials`, `Cadastros`)? Todos devem mudar?
- Volume/performance: lista server-side (DataTables) ou carga total? Consulta usa `NOLOCK`; aceita leitura suja?

## 4. Riscos por camada

| Camada | Riscos principais |
|---|---|
| View/JS (`.cshtml`, `Content/js`) | Contrato JSON inconsistente (`sucesso`/`success`); script inline por tela; menu no `localStorage`; sem anti-forgery; TinyMCE/DataTables do `node_modules` versionado |
| Controller | Regra de negócio dentro do controller (Empresas/Logins); autorização só no `Index`; `new BLL` sem `using`; exceções engolidas |
| BLL | Compartilhada com Api/JobHost/net10; validações vazias (`ValidarAdd`...); fluxos sem transação (Route 53, tenant, MyBI) |
| DAL/SQL | SQL inline/concatenado; `NOLOCK`; conexão para tenant montada manualmente; `#if NETFRAMEWORK` nos usings; compatibilidade net48/net10 |
| Domain/Infra (EF) | Coluna nova sem Map/DbSet; `AutomaticMigrationsEnabled=false`; entidade compartilhada |
| Script SQL | Não idempotente, `GO`, falta ROLLBACK, nome fora do padrão, esquecer `<Content>` no csproj, `(EMPRESAS)` rodando em todos os clientes |
| Api | Contratos DTO com o CRM; endpoints sem autenticação (`SignIn`, `JobHost/RestartService`); GET destrutivos; erro de negócio = HTTP 500 |
| JobHost/filas | Falta linha em `QueueProcess`; enum `QueueType` desalinhado; locks órfãos; retentativas (3 por padrão) |
| Config/infra | Parameter Store por ambiente; segredos versionados; não commitar troca de `AWSParameterStoreEnvironment` |

## 5. Impacto cruzado com o CRM (checklist)

- Mexeu em **login/senha/bloqueio**: `BLLLogin`, `ControleBloqueioConta`, `Logins/RealizarLogin` da Api e o `LoginController` do CRM (`SMark/Web`). Testar login CRM e BackOffice.
- Mexeu em **menus/permissões**: lembrar que o CRM filtra por `Menus.path`/`Menus.menu` (comparações literais em `BaseController` etc.) e guarda menus na sessão; script em `Menus` deve ser idempotente e por nome+path.
- Mexeu em **empresas/licenças/módulos**: `Empresas`, `ModulosSistemaEmpresas`, enum `ModulosDoSistema` do CRM e `PermissaoUtil.VerificarModulos`; Api `Empresas`/`Licencas`.
- Mexeu em **campos dinâmicos** (`SmarkEntidades`/`SmarkCampos`): Api `Campos` alimenta o CRM; scripts BACKOFFICE do tipo "Assistente-Add-..." mexem nessas tabelas.
- Mexeu em **fila**: CRM enfileira por `QueueService`; JobHost executa; BackOffice só administra.
- Mexeu em **DTO/rota da Api**: ajustar `src/SMark/ServicesBackOffice/Api/v1`, SMark.Api, JobHost e (fora de escopo, mas existem) Smark.UI/Smark.BackOffice.Api.
- Mexeu em **tabela do tenant**: script `(EMPRESAS)` + entidade/Map no CRM (`src/SMark/SiteExpress.SMark.Domain/Infra`).
- Mexeu em **BLL/DAL compartilhados**: build `Smark_SolutionCompleta.sln` (inclui Api, JobHost, WebForms, MVC) e conferir `net10.0` dos projetos multi-target.

## 6. Validação manual mínima (sem testes automatizados)

1. Build da solution (`Smark_SolutionCompleta.sln`, Debug) sem erros.
2. Subir MVC (49981) e login com usuário do BackOffice; passar pela(s) tela(s) alteradas: listar, criar, editar, excluir, filtros.
3. Se tocou a Api/CRM: subir BackOffice.Api (1999) + CRM; logar no CRM, abrir menus, ação afetada.
4. Rodar o script SQL **duas vezes** (idempotência) e o ROLLBACK, conferindo com `SELECT` antes/depois.
5. Para fila: enfileirar item, ver em `Ferramentas/ListaJobhosts`, processar no JobHost local, conferir `QueueMessages`.
6. Registrar o que não pôde ser testado (ex.: Route 53, MyBI, e-mail) como risco do plano.
