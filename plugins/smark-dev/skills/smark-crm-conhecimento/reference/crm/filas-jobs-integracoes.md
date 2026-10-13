> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Filas, JobHost, BackOffice.Api e integrações externas

Objetivo: descrever o caminho de um processamento assíncrono (CRM -> BackOffice.Api -> fila -> JobHost), o ciclo do serviço Windows e onde ficam as integrações com terceiros.

## CRM -> BackOffice.Api (proxy)
- `SMark/ServicesBackOffice/Api/ServiceBaseApi.cs` (abstrata): header Basic com base64(`subdominio\login:senha[:tipo]`), URL = `AppSettings.GetRequired("BackOfficeServicesApiBaseUrl")` + `ApiController` + action; verbos `TypesVerbs.Get/Post/Delete`; chamadas síncronas (`.Result`); log opcional em tabela (`ativaLog`).
- Serviços em `Api/v1/*Service.cs` (cada um define `ApiController = "<X>/"`): `LoginService` (RealizarLogin, Logout, ForceLogoutSession, Add/Update/DeleteLogin, troca/expiração de senha, blacklist, `GetAtivoAndLicenciadoByIds`...), `GrupoService` (grupos e permissões), `EmpresasService` (GetEmpresa, CreateTrial, plano do cliente...), `LicencasService`, `CamposService`, `NotificacoesService`, `QueueService`, `EnderecoService`, `CIDService`, `CnaeService`, `RelatorioEngajamentoService`, `UsoFuncionalidadesService`, `AuthenticationService`.
- Lado servidor: `Services/SiteExpress.SMark.BackOffice.Api/v1/Controllers/*Controller.cs` (rota `api/v1/{controller}/{action}/{id}`). Retorno de erro padrão: `HttpErroResponseMessage` com 500. Autenticação: ver `seguranca-sessao-config.md` (inclusive controllers sem atributo).
- Para nova capacidade que mora no BackOffice: action no controller da API + método no `*Service` do proxy + wrapper na BLL (`BLLLoginDTO` em `BLL/Services/Login`). A UI só funciona com a BackOffice.Api rodando.

## Fila de processamento
Caminho completo:
1. Controller (ex.: `Relatorios/AnaliseDeFechamento`, `ImportarDados`, `GeradorProposta`, `AssistenteRelatorio/Relatorios`) monta `QueueMessageDTO` (`type` = `QueueType`, `data` = JSON do filtro) e chama `BLLQueueFile.Enqueue(...)` (`BLL/Queue/BLLQueueFile.cs`).
2. `Enqueue` grava `QueueFile` no banco do tenant (status `Aguardando`), injeta `idQueueFile` no JSON do filtro (um `if` por tipo; o ramo `else` desserializa em `RelatorioModeloDTO` e reserializa, então tipos novos com DTO próprio precisam de ramo próprio, não verificado o efeito em outros tipos) e chama `QueueService.Enqueue` na BackOffice.Api. Importações usam a sobrecarga que só devolve o id da mensagem.
3. `QueueController.Enqueue` (BackOffice.Api) procura `QueueProcess` por `type` (`QueueType`) e grava `QueueMessage` (BACKOFFICE) com prioridade do processo; retorna `{ id }`.
4. JobHost: `QueueMessageProcess` reivindica lotes (`BLLQueueMessage.ReclamarProximos(lote, worker, timeoutMin, maxTentativas)`), resolve a classe por **`QueueProcess.classe`** (nome da classe, case-insensitive) no dicionário `MapaProcessos` (todas as subclasses não abstratas de `QueueBaseProcess` do assembly) e instancia `(queueMessage, idEmpresa)`.
5. `ContainerComponentQueueProcess` agrupa por (empresa, login, prioridade, tipo) e executa um item por grupo por vez; duplicados do mesmo grupo no mesmo lote vão para uma segunda passada; paralelismo por `IsParallelQueue` + `TotalDeQueuesParalelos`; timeout por item `QueueTimeoutPorItemMinutos` (padrão 20). Item com status `Cancelado`/`Pausado` é pulado.
6. `QueueBaseProcess.Start`: sucesso -> `FinalizarSucesso`; erro -> `FinalizarFalha(id, msg curta com errorId, maxTentativas)` com backoff e dead-letter; cancelamento por timeout deixa o item em `Processando` até virar lock órfão.
- Enums: `QueueType` (`Global/General/Enum/QueueType.cs`): ContatosDaConta 1, ExportAssistenteContas 3, ExportAssistenteOportunidades 4, RessarcimentoDespesas 5, GestaoDePropostas 6, Georeferenciamento 7, MapeamentoEnderecos 8, Importacao 9, Notificacao 10, ResultadoPorColaborador 11, ContatosEfetuados 12, GeradorDeProposta 13, ExportAnaliseFechamento 14, ImportacaoDeProdutos 15, ExportRelatorioTarefas 16, ExportRelatorioAtividadeRealizada 17, ImportacaoDeBeneficiarios 18, ImportacaoDeComboBox 19, ExportTaxaDeConversao 20. `QueueStatus` (Normal 1, Pausado 2, Cancelado 3, Erro 4) e `QueueStatusExecucao` (Pendente 0, Processando 1, Concluido 2, Erro 3, Morto 4) em `BackOffice/Domain/Enumeradores`.
- Classes de processo (`SiteExpress.SMark.Services/Process/QueueProcess`): `QueueExportAnaliseFechamento`, `QueueExportAssistente` (+ pasta `Assistente`), `QueueExportAtividadeRealizada`, `QueueExportContatosDaConta`, `QueueExportGestaoPropopstas` (grafia), `QueueExportRelatorioTarefas`, `QueueExportRessarcimentoDespesas`, `QueueExportTaxaDeConversao`, `QueueContatosEfetuados`, `QueueResultadoColaborador`, `QueueGeradorDeProposta`, `QueueImportacao`, `QueueImportacaoDeBeneficiarios/ComboBox/Produtos`, `QueueNotificacao`.
- **Processo novo de fila**: valor em `QueueType`, classe `Queue<X> : QueueBaseProcess` (`StartQueueProcessAsync`), linha em `QueueProcess` (script `(BACKOFFICE) Insert-QueueProcess-...` com `type` e `classe` = nome exato da classe; há exemplos em `Scripts por Sprint`), ramo no `Enqueue` se o JSON do filtro precisar de `idQueueFile`.
- Colunas de controle de execução em `QueueMessages` (`statusExecucao`, `lockedBy`, `lockedAt`, `tentativas`, `proximaTentativa`) vêm do script `Scripts por Sprint/3.32/09 - (BACKOFFICE) Add-Columns-ControleExecucao-QueueMessages-MIGRATE.sql`.

## JobHost (serviço Windows)
- Projeto `SiteExpress.SMark.JobHost` (`Program.cs`, `JobHost.cs`, `Synchronizer.cs`, `InstallerJobHost.cs`). Serviço `JobHostServices`, conta `NetworkService`. Não é Topshelf. Em `DEBUG` roda como console (thread `JobHostProcess`).
- Start: handlers globais de exceção; `IniciarParameterStore` (BackOffice.ServicesProxy.AWS.SSMService, sem checar `UsarParameterStore`; falha só é logada); `ValidarConfiguracoes` (chaves críticas, só loga); `OnStart`: `WindsorFactory.Iniciar()` (BackOffice.Ioc), `RecuperarLocksOrfaos` da própria máquina, loop `DoWorkAsync`.
- Loop: `Synchronizer.StartProcessAsync` a cada `ExecutionFrequencySeconds` (fallback `ExecutionFrequency` em segundos; padrão 60 s). Dentro de `WindsorFactory.AbrirEscopo()` monta `ContainerComponentProcess` conforme flags do `App.config`: `isQueueMessageProcess` (a fila), `isAddEmpresaTrialProcess`/`isRemoveEmpresaTrialProcess` (ciclo de vida de empresas trial; scripts em `SiteExpress.SMark.Services/Sql`), `isNotificationEmail` (e-mails de trial 3/5/9/11/15 dias), `isBIGocil` (processo de BI agendado por dia/hora, `BIGocilProcess`). No `App.config` versionado: fila ligada, trial/e-mail/BI desligados.
- `OnStop`: cancela token, espera até 110 s. `HealthCheck.RegistrarCiclo()` a cada ciclo. Chaves de fila: `QueueClaimLote`, `QueueLockTimeoutMinutos`, `QueueMaxTentativas`, `QueueTimeoutPorItemMinutos`.
- `RedisSessionManagerProccess` (limpeza de sessões Redis) existe, mas **nenhum código chama `Synchronizer.RunOnceTime()`** (verificado): está inerte.
- Deploy: `publish-jobhost.csproj`, `safeServiceStop/Start/Delete.bat`, `InstallUtil`. Configs por ambiente: `App.aws_dev.config`, `App.aws_hlg.config`.
- `src/RELATORIO_ALTERACOES.md` (11/08/2026) descreve o plano de correção do JobHost (intervalo de ciclo, claim atômico, dead-letter, dispose). **Divergência**: ele afirma remoção das credenciais fixas do `SSMService` e `Encrypt=True` em `DbConfiguration`, mas o código atual ainda tem credenciais literais nas duas cópias de `SSMService.cs` e `Encrypt=False`. Conferir o código, não o relatório.

## Hangfire (UI)
`IsHangfire=false` no `Web.config` versionado. Se `true`: cria o banco `DbHangfire` no servidor da conexão `HangfireConnection` (Parameter Store, app `CRM`) e sobe `BackgroundJobServer`. Usos: envio de e-mail em `BLLBase`, `BLLAcoesDaOportunidade`, `BLLGerenciarTarefas`, `EmailsController`; sem a flag, e-mail vai síncrono.

## Integrações externas (`SMark/ServicesBackOffice/<X>` + `Web/Areas/Integracao`)
Descrição inferida dos nomes/chaves (cada uma tem client/service próprio; erros com exceção dedicada e log4net):
| Pasta/serviço | Uso | Config |
|---|---|---|
| RDStation (+`OAuth`, Flurl) | Leads/oportunidades e contatos do RD Station, callback OAuth | `rdstation:*`, `RDStationAPIV3`, `urlRDStation*` |
| Dinamize | E-mail marketing: leads, listas, campos | `Dinamize:*` |
| Assinatura (DocuSign, ClickSign, D4Sign, Asten) | Assinatura eletrônica de propostas/documentos, webhooks de status | `docusign:*`, `clicksign:*`, `d4sign:*`, `asten:*` |
| Google / Microsoft (OAuth) | Integração de e-mail/agenda Gmail e Office 365 | `Google:*`, `Microsoft:OFFICE365:*`, `EmailIntegrationAPI:*` |
| Zoom, Zenvia | Reuniões; mensagens (callbacks de nova mensagem/status) | `Zoom:*`, `Zenvia:*` |
| Movidesk | Tickets/conversas da plataforma de atendimento | `chatSuporteMovidesk`, `plataformaAtendimento:callback` |
| Protheus (WCF em `Connected Services`) | ERP: validação de contas e propostas, integração de orçamentos | `Protheus:*` |
| Solus, Becon, Planium, UnimedCBS, IntegracaoERP, IntegracaoRE, ESales | Integrações setoriais (plano de saúde, documentos Unimed CBS, ERP, frete/ICMS) | `Solus:*`, `Becon:*`, `GerenciadorDocumentosService:UnimedCBS:*`, `Esales:*` |
| Endereco (GoogleApi, ViaCep) | Consulta de CEP/endereço | `QualApiEnderecosUsar`, `UrlViaCepApi`, `UrlGoogleMapsApi` |
| IA (`IAClient`, `IAService`) | Resumo/solicitações de IA (módulo `AssistenteIA`) | `IASolicitacoes:*`, `MarkAPI:Url` |
| Email (`EmailService`, `EmailSmtpService`), AWS (`S3BucketService`, `SSMService`), `Util/LogService` | E-mail do sistema, S3, Parameter Store, logs | `Send_Mail_*`, `AWSBucketName` |
- Integração de e-mail do usuário: `BaseController.AutenticarIntegracaoEmail`/`AutenticarAPIQueues` falam com `QueueCoreAPI` e `EmailIntegrationAPI` (serviços externos ao repo).
- Regra de uso: URLs/ids por cliente vão em `Configuracoes` (`NomeConfiguracao`); credenciais de plataforma no Parameter Store. Configuração de integração por cliente aparece em `Configuracoes/DeSistema` e `MapeamentoCamposIntegracao`.
- `SiteExpress.SMark.Api` (Web API legada, 43 controllers: Mobile, Trial, IntegracaoERP, IntegracaoSite, Renner, FFM, Alertas, `IntegracaoGmail`) atende apps e integrações de entrada; está na solution e usa as mesmas BLL/Ioc.
