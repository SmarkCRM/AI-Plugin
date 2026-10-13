> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Areas da UI e módulos de negócio

Objetivo: dizer, para cada Area de `src/SMark/Web/Areas`, o que ela faz, quais controllers e BLLs principais e onde achar a tela. Descrições vêm dos nomes dos controllers, das BLLs que eles usam e de leitura pontual; o detalhe de cada regra deve ser conferido no código.

## Convenções de localização
- Rota de Area: `{Area}/{controller}/{action}/{id}` (`<Area>AreaRegistration.cs`). Rotas fora de Area em `App_Start/RouteConfig.cs` (ex.: `Cadastro/{action}`, `TrocarSenha/{action}/{hash}`, `AceiteBasesLegais/{action}/{hash}`).
- Tela `X`: `Areas/<Area>/Controllers/XController.cs` -> `Areas/<Area>/Views/X/Index.cshtml` (+ `_Form.cshtml` em modal) -> `Areas/<Area>/Scripts/X.js` (+ `X.min.js`). Objeto JS: `se<X>`.
- Controller `X` usa `BLL<...>` do mesmo tema; BLL em `SiteExpress.SMark.BLL/<Pasta>`; DAL espelhada em `SiteExpress.SMark.DAL/<Pasta>`. Pastas de BLL/DAL: `Comercial`, `BancodeDados` (d minúsculo), `Configuracoes`, `Ferramentas`, `Home`, `Relatorios`, `AssistenteRelatorio`, `Tarefas`, `Integracao`/`Integracoes`, `Utilitarios`, `Queue`, `Seguranca`, `Services`.
- Controllers fora de Area em `Web/Controllers`: `LoginController`, `TrocarSenhaController`, `CadastroController` (trial/cadastro), `ConfiguracoesController`, `DeclaracaoDeSaudeController`, `DocumentosBeneficiarioController` (links públicos para beneficiário), `AceiteBasesLegaisController` (LGPD), `EmailsDoSistemaController`, `DownloadController`, `RedirectController`, `InfoController`, `ErrorController`; bases `BaseController` (3.083 linhas), `BaseAsyncController`.
- Contagem de controllers (arquivos `*Controller.cs`): Comercial 26, BancoDeDados 43, Relatorios 27, Ferramentas 23, Integracao 13, Home 7, Configuracoes 7, AssistenteRelatorio 5, Tarefas 1 (total 152).
- Controllers gigantes (cuidado ao editar): `OportunidadesController` 4.749 linhas, `ContasController` 3.529, `TarefasController` 3.404, `HomeController` 3.316, `PainelDeBeneficiariosController` 3.110, `FiltrosController` (AssistenteRelatorio) 3.033, `PainelDeDeclaracaoDeSaudeController` 2.931, `DeUsuarioController` 2.860.

## Comercial (26) — núcleo do CRM
Oportunidade e pós-venda são a **mesma tabela** `Oportunidades`, separadas por `tipo` (`TipoRelacionamento`: Nenhum 0, Conta 1, Oportunidade 2, PosVenda 3, Mercado 4, Lead 5).
| Controller | Para que serve | BLL principais |
|---|---|---|
| OportunidadesCards | Painel de oportunidades (cards, kanban, grid, filtros, totais de propostas) | BLLOportunidades, BLLFunilVendas, BLLTiposDeOportunidades |
| Oportunidades | Cadastro/edição completa da oportunidade | BLLOportunidades, BLLPropostasDaOportunidade, BLLMoedas |
| PosVendasCards / PosVendas | Painel e cadastro do pós-venda | BLLOportunidades, BLLClassesClientes |
| ContasCards / Contas | Painel e cadastro de contas (PF/PJ/CAEPF, `TipoPessoa`) | BLLContas, BLLOportunidades |
| Contatos, Enderecos, Emails | Contatos da conta, endereços (CEP/mapas), e-mails enviados/templates | BLLContatos, BLLTelefones, BLLEnderecos, BLLEmail |
| AcoesDaOportunidade, AgendamentosDaOportunidade, ProximoPassoDaOportunidade | Ações realizadas, recontato/visita/agendamento (`TipoAgendamento`), próximo passo | BLLAcoesDaOportunidade, BLLAgendamentosDaOportunidade, BLLAgenda |
| PropostasDaOportunidade, ProdutosServicosDaPropostaDaOportunidade, GeradorProposta | Propostas, itens, geração de documento (via fila `GeradorDeProposta`) | BLLPropostasDaOportunidade, BLLGeradorProposta, BLLQueueFile |
| DespesasDeAcoesDaOportunidade, AnaliseDeRessarcimentos | Despesas e ressarcimento | BLLDespesasDaAcao |
| Metas | Metas de venda/produto | BLLMetas, BLLMetaVendaTotal |
| Estatistica, PainelDeRelacionamentos | Estatísticas e painel de relacionamento (recontato) | BLLOportunidades |
| Beneficiarios, BeneficiariosUploadArquivos, PainelDeBeneficiarios, PainelDeDeclaracaoDeSaude, SimuladorPorFaixasEtarias | Vertical de plano de saúde: beneficiários, upload, declaração de saúde, simulador por faixa etária (módulos `DeclaracaoDeSaude`, `SimulacaoDePlanoDeSaude` em `ModulosDoSistema`) | BLLBeneficiarios, BLLHistoricoDeclaracao |
| BaseOportunidades | Base abstrata + actions auxiliares (menu de ações, info de conta/contato) | — |

### Mapa de `BLLOportunidades` (2.689 linhas; DAL `DALOportunidades` 8.492 linhas)
- Ctors: `()` (resolve por Windsor), `(IUsuarioLogadoSession, ...)` e `(ISMarkContext, bool isDispose)` (JobHost/threads).
- Escrita: `AddOportunidade`, `Update` (sobrecarga longa com recontato/colaboradores/informados/genéricos/encerramento), `UpdateReaberturaOportunidade`, `UpdateAvaliacao`, `Delete`, `DeletePosVenda`, `VincularOportunidade`, `Adicionar/RemoverTipoOportunidade`, `TransferirResponsavel`, `TransferirOportunidades*` (troca em massa de funil/origem), `AtualizarFunilDeVendasKanban`/`ValidarAtualizarFunilDeVendasKanban`, `GerarHistoricoFunil`.
- Validação: `ValidarAdd/ValidarUpdate/ValidarDelete`, `ValidarOportunidade` (privado, regras de encerramento/carteira/campos), `ValidarPosVenda`, `VerificarCamposObrigatorios`, `ValidarRetrocederEtapaDoFunil/Origem`.
- Pesquisa: `Pesquisar` (cards/kanban), `PesquisarGrid`, `PesquisarCount`, `PesquisarTotalPropostas`, `PesquisarFiqueDeOlho*`, `PesquisarTransferenciasCarteiras*`.
- Indicadores: `ObterDadosGrafico*`/`ObterDadosTabela*` (dashboard Home), `ObterQtdRecontatos*`/`ObterQtdAgendamentos*` (atrasados/hoje/semana, oportunidade e pós-venda), aniversários.
- Há métodos para API mobile (`RetrieveSimpleApi`, `AbrirOportunidade`) usados pela `SiteExpress.SMark.Api`.

## BancoDeDados (43) — cadastros auxiliares (CRUD jqGrid + modal)
Padrão `Index/List/Create/Edit/Delete`. Principais: Acoes, FunilDeVendas (etapas), Origens, MotivosDeSucesso, ImpedimentosVenda, ProbabilidadesSucesso, Receptividades, TiposDeOportunidades, Concorrentes, Cotacoes/MoedasCotacoes, Despesas, FaixasDeDescontos, FaixasEtarias, FamiliasDeProdutos, ProdutosServicos, UnidadeDeProduto, PrazosDeEntrega, ClassesClientes, Segmentos, RamosDeAtividades, AreasDeAtuacao, Cargos, Regioes, Nacionalidades, GruposEmpresariais, CategoriasEmpresariais, CategoriasAnexos, Interlocutores, EnderecosEmpresa, Tags, Roleta (distribuição por rodízio, com histórico/auditoria), AdministraCeps (transferência de CEPs entre matrículas), LoginHorariosAcesso, BasesLegais (LGPD), Genericos (campos personalizados), Templates (templates de e-mail, com tokens), TarefaCategoria, TarefaStatus, TiposDeTarefas, TitulosWorkflow, ComboBox, ProdutosServicosList.
Acesso a tela depende de linha em `Menus` (banco BACKOFFICE). Texto do submenu vem de `Menus.menu`, não de resx.

## Tarefas (1)
`TarefasController`: tarefas com workflow, status, comentários, "manter informado", anexos, tipos de relacionamento. BLL `BLLGerenciarTarefas` (tem `BackgroundJob.Enqueue` para e-mail quando `IsHangfire=true`). Entidades `Tarefa*`, `Workflow*`.

## Relatorios (27) e AssistenteRelatorio (5)
- Relatorios: relatórios prontos. AnaliseDeFechamento, AnaliseFunil, AtividadeRealizada, DeVendasRealizadas(+Natureza), DeResultados(+Natureza), EvolucaoDeVendas, TaxaConversao, ConversaoSobrePerdido, RankPorOrigem, TotalTipoOportunidade, VisitasComerciais, GestaoDePropostas, Ressarcimento, ClientePorConcorrente, ResultadoColaborador, Internacional, RelatorioTarefas, Acompanhamento, Agendamento(s), Anexos, Pivot, BaseRelatorios, Assistente, Teste. BLLs em `BLL/Relatorios`; DAL herda `DALBaseRelatorios`. Exportações pesadas vão para fila (ver `filas-jobs-integracoes.md`).
- AssistenteRelatorio: relatórios montados pelo usuário (Campos, Categorias, Filtros, Relatorios, Viewer). Entidades `RelatorioModelo*`, `RelatorioFormula*`. BLL `BLL/AssistenteRelatorio`.

## Configuracoes (7)
DeSistema (configurações por empresa, tabela `Configuracoes` via `NomeConfiguracao`), DeUsuario (preferências, e-mail/assinatura do usuário), CamposObrigatorios (regras por `PaginasComRegras` e etapa do funil), ColunasOportunidade/DashboardCampos (`MapListagem`), MapGenericos/MapeamentoCamposIntegracao (campos personalizados e mapeamento para integrações).

## Ferramentas (23)
Usuarios, GrupoUsuarios (permissões, permissões especiais, horários), SupervisoresDeCarteiras, SupervisoresCanais, Canais, Corretoras, TransferenciasCarteiras (`TipoTransferencia`: Conta, Oportunidade, DataRecontato), Unificacoes (de/para de entidades duplicadas), Segmentacoes + SegmentacaoAcompanhamento (seleção em lote: atualizar campos, clonar, criar, encerrar oportunidades, criar pós-vendas), ImportarDados/ImportarProdutos/ImportarBeneficiarios/ImportarComboBox (via fila), ExportarDados, AtualizarOportunidades, Upload, Logs, Licenciamentos (usa BackOffice.Api), HorariosAgendamento, GerenciamentoDeConsentimentos (LGPD), Gatilhos (`BLLQueues`), IA (`BLLSolicitacaoIA`).

## Home (7)
Home (dashboard/widgets; o logout está em `HomeController`, o login em `Controllers/LoginController`), Agenda, FiqueDeOlho (pendências: recontatos atrasados, etc.), Notificacoes, Widgets (`BLLWidget*`), ConversasPlataformaAtendimento (Movidesk: vincula conversas a contas/oportunidades), TestRDStation (teste).

## Integracao (13)
Controllers que recebem OAuth/webhooks (muitos `[AllowAnonymous]`) e configuram integrações: RDStation, Dinamize, Google, Microsoft, Zoom, DocuSign, ClickSign, D4Sign, Asten, Zenvia, Reev, Generico, GerenciadorDocumentos. Clientes HTTP em `ServicesBackOffice/<X>`. Detalhe e lista de endpoints anônimos em `filas-jobs-integracoes.md`. Há também `Areas/Integracao/WebServices/Protheus` e `Teste.asmx`.

## Módulos de negócio (visão transversal)
- Funil/etapas: `FunilVenda` (+`HistoricoFunilDeVenda`); regras de retroceder etapa via `TipoPermissaoEspecial`.
- Status: `StatusOportunidade` {Todos 0, EmAndamento 1, Standby 2, Finalizada 3}; resultado `ResultadoOportunidade` {EmAberto 0, VendaRealizada 1, VendaNaoRealizada 2}.
- Campos personalizados ("genéricos"): `GenericoAba/Campo/CampoItem/CampoValor`; parse em `BaseController.ParseGenericosCamposValores`.
- Tabelas dinâmicas: `TabelaDinamicaConfiguracao/Linhas` (actions no `BaseController`).
- Módulos contratados: `ModulosDoSistema` {PainelDeContas 1, PainelDeOportunidade 2, PainelDePosVenda 3, DeclaracaoDeSaude 4, SimulacaoDePlanoDeSaude 5, AssistenteIA 6, DashboardSmarkBI 7}.
- Multi-tenant: um banco por empresa (ver `banco-de-dados.md`).
- Tempo real: SignalR em `Web/Services/SignalR` (hubs de declaração de saúde: agendamento do beneficiário e liberação pelo médico), com backplane Redis.
