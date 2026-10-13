# BackOffice MVC — módulos, áreas e telas
> **LEGADO em descomissionamento:** o BackOffice 4.8 foi migrado para o repositório `smark-backoffice` (ver `../backoffice-novo/`). Consulte este arquivo só para dúvidas do CRM que dependem do código 4.8 (BackOffice.Api, BLL/DAL compartilhados, banco BACKOFFICE).

> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

Objetivo: mapa de Areas/Controllers/Views do `SiteExpress.Smark.BackOffice.MVC.UI`, o que cada tela faz e de onde vem a lista de menus. Base: `src/BackOffice/SiteExpress.Smark.BackOffice.MVC.UI` (abreviado `MVC/`).

## 1. Roteamento e navegação

- Rota raiz: `{controller}/{action}/{id}` → `Home/Index` (`App_Start/RouteConfig.cs`). Áreas registradas por `AreaRegistration`: `Cadastros/…`, `Ferramentas/…`, `IA/…` (padrão `Area/{controller}/{action}/{id}`, action default `Index`).
- Fluxo de entrada: `Login/Index` (view) → JS `Content/js/Login.js` chama `GET /Login/LogarUsuario?usuario=&senha=` → grava o JSON de retorno em `localStorage['dados']` → redireciona para `Home`.
- **O menu lateral é montado no navegador**: `Views/Shared/_Layout.cshtml` lê `localStorage['dados']`, e para cada item de `dados.menu[0].menus` (nome + path vindos de `Menus`/`MenusGrupo`) cria um `<li>` em "Ferramentas" ou, se o path começa com `../IA/`, em "IA". Se `admin` = true, acrescenta o link fixo "Usuários/Grupos Backoffice" (`../Ferramentas/Permissoes`). Sem `dados` no localStorage o layout redireciona para `/Login`.
- Quais itens aparecem = linhas de `Menus` com `idPai = appSetting BackOfficeIdMenuPai` ligadas aos grupos do usuário (`MenusGrupo`) — ver `LoginController.LogarUsuario`. **Menu novo = (1) controller/view, (2) linha em `Menus` via script SQL BACKOFFICE, (3) vincular ao grupo** (tela de Permissões ou script). Não existe CRUD de `Menus` no MVC; o do WebForms (`UI/Views/Configuracoes/Menus.aspx`) é o único (se ainda em uso: não verificado).
- Layouts: `Views/Shared/_Layout.cshtml` (principal, tema Gentelella), `_LayoutSenhaExpirada`, `_LayoutTrocaSenha`. Todas as áreas herdam via `_ViewStart.cshtml`.

## 2. Controllers raiz (`MVC/Controllers`)

| Controller | O que faz |
|---|---|
| `LoginController` | `Index` (tela) e `LogarUsuario` (JSON, GET). Valida via `BLLLogin.ValidaLogin`, preenche `IUsuarioLogado`, `FormsAuthentication.SetAuthCookie`, devolve menus e flag `admin`. |
| `HomeController` | Dashboard: contador de licenças (`BLLLicencas`) e documentos processados (`BLLListaJobHosts`). `About/Contact` são sobra do template. Redireciona ao login se `IUsuarioLogado.idEmpresa == 0`. |
| `ConfiguracoesController` | Troca de senha expirada (`AlterarSenhaExpirada`/`SalvarSenhaExpirada`) e redefinição pública (`RedefinirSenha`/`SalvarRedefinirSenha`, `[AllowAnonymous]`). Valida força da senha (`ValidadorSenhaSeguranca`). |
| `GeradorSenhaController` | Gera senha segura (`GeradorSenhaSegura`) para as telas de usuário. |

## 3. Area `Ferramentas` (núcleo do BackOffice)

| Controller (rota `Ferramentas/<nome>`) | Tela / função | BLL principais |
|---|---|---|
| `Empresas` (1,5 mil linhas, o maior) | Lista de empresas com filtros (ativo, trial, plano, demo, módulo); editar; ativar/inativar; "excluir" (é soft: `ativo=false`); **Transformar trial em cliente** (define subdomínio, licenças, renomeia o banco `smark_<subdominio>_<id>`); módulos/funcionalidades por empresa; Smark BI (usuários e abas MyBI, sincronizar permissões); uso de funcionalidades | `BLLEmpresas`, `BLLLogin`, `BLLServidores`, `BLLUsuariosSmarkBI`, `BLLAbasMyBI`, `BLLUsoFuncionalidades`, `MyBIService` |
| `Trials` | Lista trials, transformar em cliente e `CriarTrial` (POST a uma API externa de trial, URL fixa no controller) | `BLLEmpresas`, `BLLLogin` |
| `GerenciarUsuarios` | Usuários e grupos **dos clientes** (CRM): listar, remover, formulário de grupo (menus/permissões/permissões especiais), formulário de login, reset de senha, desbloqueio de tentativas | `BLLLogin`, `BLLGrupos`, `BLLMenus`, `BLLPermissoes`, `BLLLoginTentativaAcessos` |
| `Permissoes` | Mesmas operações, mas para usuários/grupos do **próprio BackOffice** (`Aplicacao = Backoffice`). Versão mais antiga e duplicada do código acima | idem |
| `Licencas` | Painel de licenças (totais, por empresa) | `BLLLicencas` |
| `GerenciarLicencas` | Solicitações de aumento/redução de licença feitas pelo cliente: aprovar/rejeitar, envia e-mail | `BLLSolicitacaoLicencas`, `BLLHistoricoEmailsDoSistema`, `BLLBlackListEmails` |
| `ModulosSistema` | Lista de módulos (`ModulosSistema`) e relacionamento com empresas | `BLLEmpresas` |
| `Avisos` | CRUD de avisos para clientes; editor TinyMCE; publicar = grava notificações dentro do banco de cada empresa alvo (`PublicarAvisoCrm`) e/ou enfileira e-mail via API de filas externa (`QueuesAPI.EnqueueSQSAvisoEmail`); acompanhamento de e-mails | `BLLAviso`, `BLLAvisoEmails` |
| `Onboards` | Cadastro de onboards (conteúdo de boas-vindas) e empresas-alvo | `BLLOnboard` |
| `ListaJobhosts` | Fila de processamento (`QueueMessages`): listar, pausar/cancelar/retomar item; status/start/stop/restart do serviço Windows do JobHost (`ServiceController`) | `BLLListaJobHosts`, `BLLQueueMessage` |
| `Auditoria` | Auditoria de consultas de CNPJ (`AuditoriaConsultaCNPJ`) por empresa/login/data | `BLLAuditoriaConsultaCNPJ` |
| `RelatorioEngajamento`, `RelatorioUsoFuncionalidades`, `Funcionalidades` | Relatórios de engajamento e uso de funcionalidades por empresa | `BLLRelatorioEngajamentos`, `BLLUsoFuncionalidades`, `BLLFuncionalidades` |

Views por controller em `MVC/Areas/Ferramentas/Views/<Controller>/` (parciais `_Form*.cshtml` abertas em modais). ViewModels em `Areas/Ferramentas/Models`.

## 4. Area `IA` (controle de tokens de IA)

| Controller | Função |
|---|---|
| `TokensIa` | Limite de tokens de IA por empresa (`Empresas.limiteTokensAI`), ajuste com auditoria (`AuditoriaLimiteTokensAI`), uso por empresa |
| `ControleTokensIaPorEmpresas` | Relatório de consumo por empresa/período (`LogTokensAI`) |
| `ControleTokensIaPorUsuarios` | Relatório de consumo por usuário/período |

BLL: `BLLTokensIa`; DAL `DALTokensIa` (SQL Dapper direto no BACKOFFICE).

## 5. Area `Cadastros` (resquício)

`Cadastros/Empresas` (Index/Pesquisa/Edit) e `Cadastros/Usuarios` (`RecuperarUsuarios`). Sem verificação de acesso e provavelmente supersedidas por `Ferramentas`; se são alcançáveis pelo menu atual: (não verificado).

## 6. Padrão de tela (como as views funcionam)

- `Index.cshtml` com **DataTables server-side**: `$('#tabela').DataTable({ ajax: ... })` apontando para a action `List(DTParameters param, ...)` que devolve `DTResult<T>` (Domain `DTO/DataTables/DataTableDTO.cs`). Filtros extras como parâmetros da querystring.
- Criar/editar/excluir abrem **parciais em modal Bootstrap** carregadas por `$.ajax` (GET devolve `_Form*`), o submit é POST e a action responde `Json(new { sucesso = bool, message = "..." })` (alguns usam `success`/`mensagem`/`error`, não há contrato único).
- Notificações com PNotify; selects com select2; datas com daterangepicker/moment; relatórios com Chart.js/Flot (tema Gentelella em `node_modules/gentelella`).
- Scripts específicos de tela ficam inline na própria `.cshtml` (seção `scripts`) ou em `MVC/Content/js` (`Principal.js`, `Menu.js`, `Login.js`) e `MVC/Scripts/Home`.

## 7. Telas do WebForms legado (`src/BackOffice/UI/Views`)

`Instancias/` (Empresas, Usuários, Licenças, AtribuirPermissoes), `Configuracoes/` (**Menus**, Servidores, Conteúdos), `Suporte`, `Log`, `Main/Login|Home`, `Migrations/RunMigrations` (usado em deploy). Handlers `.ashx` em `UI/Handler/*`. Login diferente do MVC: usuário no formato `subdominio\login`. Não estenda; use o MVC, exceto para o runner de migrations.
