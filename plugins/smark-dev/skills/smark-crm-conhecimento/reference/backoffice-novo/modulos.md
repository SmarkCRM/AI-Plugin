# BackOffice novo — módulos e features

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: catálogo de cada módulo existente (rota de API, policy, tela, tabelas, regras notáveis, ADR). Use para responder "onde fica X" e "o que muda se mexermos em Y".

Convenção: API `/api/<modulo>` em `backend/src/SmarkBackOffice.Api/Endpoints/<Modulo>Endpoints.cs`; serviço/DTOs em `Application/<Modulo>/`; EF/repositório em `Infrastructure/Persistence/{Configurations,Repositories}/`; tela em `frontend/src/features/<modulo>/`. Policies: ver `autenticacao-e-autorizacao.md` ("menu" = acesso por `Menus.path`).

## Quadro geral

| Módulo | API | Policy | Tela (rota SPA) | ADR |
|---|---|---|---|---|
| Autenticação | `/api/auth` (login, logout, me) | login anônimo; `me` só sessão | `/login` | 0002, 0006 |
| Home / Dashboard | `/api/dashboard` | só sessão (exceção intencional); reprocessar = admin | `/` | 0009-historico |
| Avisos | `/api/avisos` | `Avisos` (menu) | `/avisos`, `/avisos/novo`, `/avisos/$id`, `/avisos/$id/acompanhamento`, `/avisos/leituras` | 0008-avisos, 0009-modernizacao, 0012 |
| Empresas | `/api/empresas` | `Empresas`; escrita `EmpresasEditar`; MyBI global `EmpresasMyBiAdmin`; servidores = admin | `/ferramentas/empresas` | (sem ADR próprio; 0006 p/ menu) |
| Usuários de clientes | `/api/usuarios-clientes` | `UsuariosClientes` | `/ferramentas/usuarios-clientes` | 0013 |
| Trials | `/api/trials` | `Trials`, `TrialsCriar`, `TrialsConverter`, `TrialsProrrogar` | `/ferramentas/trials` | 0007-trials, 0014 |
| Rel. Engajamento | `/api/relatorio-engajamento` | `RelatorioEngajamento` (admin também passa) | `/ferramentas/relatorio-engajamento` | 0007-relatorio |
| Rel. Uso Funcionalidades | `/api/relatorio-uso-funcionalidades` | `RelatorioUsoFuncionalidades` (admin passa) | `/ferramentas/relatorio-uso-funcionalidades` | 0011 |
| Auditoria CNPJ | `/api/auditoria-cnpj` | `AuditoriaCnpj` | `/ferramentas/auditoria-cnpj` | — |
| Logs de Auditoria | `/api/logs-auditoria` | `LogsAuditoria` | `/ferramentas/logs-auditoria` | 0010 |
| Onboards | `/api/onboards` | `Onboards` | `/ferramentas/onboards` | — |
| Fila JobHosts | `/api/fila-jobhosts` | `FilaJobHosts`; `/servico/*` = `FilaJobHostsControle` (menu E admin) | `/ferramentas/fila-jobhosts` | 0008-jobhost |
| Usuários do BackOffice | `/api/permissoes` | `AdministradorBackOffice` | `/usuarios` | 0004 |
| Tokens de IA | `/api/tokens-ia` | `TokensIa`, `TokensIaPorUsuarios`, `TokensIaPorEmpresas`, `TokensIaQualquerTela` | `/ia/tokens`, `/ia/tokens/usuarios`, `/ia/tokens/empresas` | 0005 |

Contagem no contrato: 95 paths (empresas 17, avisos 15, permissoes 12, fila-jobhosts 11, usuarios-clientes 10, relatorio-engajamento 8, logs-auditoria 7, onboards 6, tokens-ia 6, trials 6, dashboard 5, auditoria-cnpj 4, auth 3, uso-funcionalidades 3, health 1).

## Detalhe por módulo

**Home/Dashboard.** `GET /contadores`, `/empresas-ativadas`, `/trials-ativos`, `/historico`; `POST /historico/reprocessar` (admin). Tabela própria `DashboardIndicadorHistorico` (índice único `(Competencia, IndicadorCodigo)`, 11 indicadores em `CodigoIndicador`). `BackgroundService` `HistoricoIndicadoresWorker` (só registrado com `Dashboard:Historico:Habilitado=true`; checa de hora em hora; roda a competência do mês anterior no dia/hora configurados, horário de Brasília, idempotente por upsert). "Empresa ativada" = `Empresas.DataVirouEmpresa` preenchida (conversão trial→cliente). Prazos do trial por config (`Dashboard:DiasTrial` 15, `DiasRemocaoTrial` 30).

**Avisos.** Entidades `Aviso`, `AvisoEmail`, `EnvioEmailAviso` (tabelas `Avisos`, `AvisoEmails`, `AvisoEmailEnvioLog`, esta só leitura). Canais: e-mail (fila SQS via Queues API, token pela Authentication API; template `Infrastructure/Email/ModeloAvisoEmail.html`) e CRM (INSERT em `Notificacoes`/`LoginContador` **no banco de cada empresa**, via `NotificacaoCrmSqlServer`). Regras: publicar só de Rascunho, revogar só de Publicado, editar só Rascunho (409 senão); publicador = usuário logado; mensagem sanitizada no servidor (HtmlSanitizer) e título/assunto com HtmlEncode; upload de imagens para S3 (`POST /imagens`, ≤ `Avisos:TamanhoMaxImagemBytes`); resumo/acompanhamento por empresa; `POST /{id}/duplicar`; `GET /leituras` consulta, em paralelo (`MaxParalelismoEmpresas`), `Notificacoes` dos tenants com limites `MaxEmpresasLeitura` (20) e `MaxLinhasLeitura` (5000) — acima, 422, nunca trunca. Não há DELETE de aviso nem status "Concluído". Auditoria: interceptor EF + evento imediato (publicar/revogar).

**Empresas.** Entidade `Empresa` (`Empresas`), `ModuloSistema`/`ModuloSistemaEmpresa`, `Servidor`, `UsuarioSmarkBi`, `AbaMyBi`. Endpoints: lista paginada com filtros (busca, ativo, trial, plano, demo, módulo, ordenação), detalhe/edição (campos de servidor só para admin), ativar/inativar/excluir, `GET|PUT /{id}/modulos`, `GET|PUT /{id}/smark-bi` + `sincronizar`, `GET /{id}/historico`, e globais MyBI (`/smark-bi/sincronizaveis`, `/smark-bi/abas`) sob `EmpresasMyBiAdmin`. MyBI: cliente HTTP `MyBiClient` (config `MyBi:*`); sem `MyBi:UrlBase` entra `MyBiClientDesligado` (falha explícita). `UsuarioBancoMyBi` cria/ajusta usuário no banco do tenant (só se `MyBi:GerenciarUsuarioBanco`). Segredos do tenant (`DataBasePassword`, senha BI) mascarados na auditoria.

**Usuários de clientes (Gerenciar Usuários 4.8).** Logins e grupos das empresas clientes (`Logins`, `Grupos`, `GruposLogin`, `LoginTentativaAcessos`). Rotas: `/empresas` (seletor + `/{id}/resumo`), `/usuarios` (lista, detalhe, `PUT`, `POST /{id}/desbloquear`), `/grupos` (lista, opções, detalhe, `PUT`). Escopo `Aplicacao IN (1,2)` **sem a empresa interna** (subdomínio `backoffice` ou com login `Aplicacao=2`). Regras: bloquear só a transição `licenciado false→true` com saldo ≤ 0 (CRM e BI); não remover o último grupo admin da empresa (422); toda gravação marca `Replicar=true` (o CRM sincroniza); telefones lidos do tenant (`Telefones`) em lote, falha do tenant não derruba a tela; "forçar troca de senha" é gravada pelo domínio mas o CRM não a respeita. Sem criar/remover usuário.

**Trials.** `GET /` (lista), `POST /` (criar = chama `PostTrial` da API do CRM via `CrmApi:BaseUrl`, depois registra "quem criou" em `TrialHistorico`), `GET /{id}` (detalhe), `POST /{id}/prorrogar` (1–7 dias somados a `Empresas.dtCriacao`, motivo ≤ 300), `GET /{id}/conversao`, `POST /{id}/converter` (`ConversaoTrialService`: valida → subdomínio único → replica logins e limpa dados no tenant (`LimparBaseTrial.sql`) → grava no BACKOFFICE em transação com `WHERE trial=1` → CNAME Route53; falha de CNAME = 502 mas conversão fica; 409 se já convertida; **não renomeia banco do tenant**, grava `DataVirouEmpresa`). Senha Zimbra gerada com `RandomNumberGenerator`. Prazo de remoção automática vem do JobHost 4.8 (`dtCriacao + 30 dias`, não editado aqui).

**Relatório Engajamento.** Só leitura de `UsoFuncionalidade`, `Funcionalidades`, `GrupoFuncionalidades`; rotas `/resumo /empresas /funcionalidades /grupos /evolucao /eventos /catalogo /empresas/{id}`. Exclusões por config `RelatorioEngajamento:NomesConfiguracaoIgnorados` e `GruposIgnorados`; período máx. `MesesMaximosPeriodo` (24). Cálculo em `Domain/RelatorioEngajamento/CalculadoraEngajamento.cs`. Sem índice em `UsoFuncionalidade` (risco de performance).

**Relatório Uso Funcionalidades.** Reusa `IRelatorioEngajamentoRepository` por composição; rotas `/funcionalidades`, `/empresas` (paginada, ordenação, filtros; com `todos=true` devolve o conjunto sem paginação para o CSV — o endpoint `/exportacao` do ADR 0011 não existe), `/resumo`. CSV é gerado no navegador (limite por config `RelatorioUsoFuncionalidades:LimiteExportacao`, padrão 10 000, 422 acima).

**Auditoria CNPJ.** Leitura de `AuditoriaConsultaCNPJ` (+ colunas `Servico`, `IsCache` adicionadas por script) — lista, detalhe, opções de empresas/usuários. Campo `Dados` é sensível (não vai para a trilha central).

**Logs de Auditoria.** Trilha central `dbo.AuditoriaEventos` (imutável: trigger + procedure de expurgo). Rotas: lista, detalhe, `/execucoes-job/{guid}`, `/opcoes*`, `/exportar` (CSV com `;`, BOM; XLSX recusado com 400; >limite = 422). Acesso só por menu exato `/ferramentas/logs-auditoria` (sem bypass de admin); acesso recusado também é auditado. Ver `backend.md` (auditoria).

**Onboards.** CRUD de `Onboards`/`OnboardsEmpresas` (lidos pelo CRM) + `GET /empresas`. Entidades auditadas (`Onboard`, junção de empresas).

**Fila JobHosts.** Lê `QueueMessages` e `QueueProcess` (BACKOFFICE) e atua em `QueueFiles` do tenant. Rotas: `/itens`, `/itens/{id}`, `/resumo`, `/opcoes`, `/servico` (status), `POST /servico/{iniciar|parar|reiniciar}` (menu E admin), `POST /itens/{id}/{pausar|cancelar|ativar}`. Controle do serviço Windows `JobHostServices` via `System.ServiceProcess.ServiceController`: só ativo no Windows com `FilaJobHosts:Servico:Habilitado=true`; senão `ServicoJobHostIndisponivel` (status "Indisponível", ações → 503). Uma operação por vez (lock em memória por instância da API). Auditoria própria em `JobHostAcoesAuditoria` + evento central (duplicação aceita).

**Usuários do BackOffice (Permissões).** `/api/permissoes/{usuarios,grupos,menus}`: CRUD de logins `Aplicacao=2` e grupos com seus menus (`MenusGrupo`), política de senha em `Application/Permissoes/PoliticaSenha.cs`. Raiz de menus = `Permissoes:IdMenuPaiBackOffice`. Tudo exige grupo `admin=1` (corrige falha do 4.8). Datas gravadas em hora de Brasília.

**Tokens de IA.** Só leitura de `LogTokensAI` e `AuditoriaLimiteTokensAI`: `/chamadas`, `/por-usuario`, `/por-empresa`, `/opcoes*`. O "ajuste de limite" do 4.8 **não** foi migrado. Menus aceitam path antigo (`../IA/...`, "contém") e novo (`/ia/tokens*`, exato).

## Itens sem ADR próprio

Empresas, Auditoria CNPJ, Onboards e Usuários de clientes (parcial) foram implementados sem ADR dedicado (apenas specs na pasta ignorada `documentation/`); a fonte de verdade é o código.

## Trials: onde está cada regra (conferido no código)

- Limite de prorrogação **por solicitação**: constante `DiasMaximoPorSolicitacao = 7` em `backend/src/SmarkBackOffice.Application/Trials/ProrrogarTrialRequest.cs` (fonte única; validator, serviço e o campo `diasMaximoPorSolicitacao` do detalhe usam ela; o front lê o valor do contrato, sem 7 fixo). Não há teto acumulado nem limite de nº de prorrogações. Testes: `ProrrogacaoTrialTests` (casos 7/8).
- Prorrogar move `Empresas.dtCriacao` (`Empresa.ProrrogarTrial`); `dtHoraCriacao` não muda; o evento vai para `TrialHistorico`.
- Prazo comercial (`Dashboard:DiasTrial`) e remoção (`Dashboard:DiasRemocaoTrial`) são calculados de `dtCriacao` no BackOffice novo. A remoção automática real é o job do JobHost (`SiteExpress.SMark.Services/Process/RemoveEmpresaTrialProcess.cs`), que usa a chave `DayTrial` do `App.config` do JobHost: **configuração separada** de `DiasRemocaoTrial`; as duas precisam ficar coerentes por ambiente (valores de hlg/prd não verificados).
- Mudar o limite por usuário (ex.: só admin passa de 7) não é possível hoje: `TrialsProrrogar` usa o mesmo menu `/Ferramentas/Trials`; exigiria menu novo (script) e segunda policy.
