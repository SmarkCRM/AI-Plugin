> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Glossário de domínio e siglas

Objetivo: traduzir termos que o usuário/work item usa para o nome real no código. `(NV)` = significado inferido, não confirmado no código.

## Negócio
| Termo | No código |
|---|---|
| Oportunidade / negociação | `Oportunidade` (`Domain/Models`), tabela `Oportunidades`, `tipo = TipoRelacionamento.Oportunidade (2)` |
| Pós-venda | Mesma tabela, `tipo = PosVenda (3)`. Outros valores: Conta 1, Mercado 4, Lead 5 |
| Painel de oportunidades (cards/kanban/grid) | Controller `OportunidadesCards`; pós-venda: `PosVendasCards`; contas: `ContasCards` |
| Situação | `StatusOportunidade`: Todos 0, EmAndamento 1, Standby 2, Finalizada 3 |
| Resultado | `ResultadoOportunidade`: EmAberto 0, VendaRealizada 1, VendaNaoRealizada 2 |
| Funil / etapa | `FunilVenda` (+ `HistoricoFunilDeVenda`), tela `BancoDeDados/FunilDeVendas` |
| Vigência | `Oportunidade.dataVigencia`; filtro de período no painel; contagem em `BLL/Comercial/Oportunidades/ContagemVigencia.cs` |
| Conta / cliente | `Conta` (`TipoPessoa`: NaoDefinido 1, Fisica 2, Juridica 3, CAEPF 4); responsável = `idLogin` da conta |
| Contato | `Contato` (pessoa da conta), com `Telefone` e `Endereco` |
| Carteira / transferência de carteira | Contas/oportunidades de um responsável; `Ferramentas/TransferenciasCarteiras`, `TipoTransferencia` {Conta 1, Oportunidade 2, DataRecontato 3}, log `TransferenciaCarteiraLog` |
| Supervisor de carteira / de canal | `SupervisoresDeCarteiras`, `SupervisoresCanais` (define "supervisionados" do usuário) |
| Canal | `Canal` (equipe/canal de venda, `BLLCanais`) (NV) |
| Recontato / visita / agendamento | `TipoAgendamento` {Recontato 1, Visita 2, Agendamento 3}; `AgendamentoDaOportunidade`, `Recontato` |
| Ação (da oportunidade) | `Acao` (cadastro) e `AcaoDaOportunidade` (registro realizado); despesas: `DespesasDeAcaoDaOportunidade`; ressarcimento: `AnaliseDeRessarcimentos` |
| Próximo passo | `ProximoPassoDaOportunidade` |
| Fique de olho | Painel de pendências (recontatos atrasados, para hoje...), `Home/FiqueDeOlho`, enum `FiltroOportunidadeFiqueDeOlho` |
| Proposta | `PropostasDaOportunidade` + `ProdutosServicosDaProposta`; gerador de documento: `GeradorProposta` (fila `GeradorDeProposta`); proposta/venda rápida: `propostaRapida`/`vendaRapida` |
| Produto/serviço, família, unidade, faixa de desconto, moeda, cotação | `ProdutoServico`, `FamiliasDeProdutos`, `UnidadeDeProduto`, `FaixasDeDesconto`, `Moeda`, `Cotacao` |
| Genéricos / campos personalizados | `GenericoAba`, `GenericoCampo`, `GenericoCampoItem`, `GenericoCampoValor`; tela `BancoDeDados/Genericos` |
| Campos obrigatórios | `Validacao` + `PaginasComRegras` (`Global/General/Enum`) + `BLLValidacoes`; tela `Configuracoes/CamposObrigatorios` |
| Origem, motivo de sucesso, impedimento, receptividade, probabilidade | `Origem`, `MotivosSucesso`, `ImpedimentosVenda`, `Receptividade`, `ProbabilidadesSucesso` (cadastros do BancoDeDados) |
| Segmentação | `SegmentacaoAcompanhamento`: seleção em lote para atualizar/clonar/criar/encerrar oportunidades |
| Unificação | `UnificacaoEntidade`/`UnificacaoItem`: de/para para unir cadastros duplicados |
| Roleta | Distribuição por rodízio (`Roleta`, `RoletaUsuario`, `RoletaRegras`, `RoletaHistorico`) |
| Gatilho | `Ferramentas/Gatilhos` (`BLLQueues`) (NV: regras automáticas) |
| Meta | `Meta*`: metas de venda, produto e unidade |
| Tarefa / workflow / manter informado | `Tarefa`, `Workflow*`, `TarefasManterInformado`, `TarefaStatus`, `TarefaCategoria` |
| Beneficiário / declaração de saúde / simulador por faixas etárias | Vertical de plano de saúde: `Beneficiarios*`, `HistoricoDeclaracao`, `SimuladorPorFaixasEtarias`, `FaixasEtarias`, CIDs |
| Base legal / consentimento | LGPD: `BasesLegais`, `BasesLegaisConsentimento`, `HistoricoBasesLegais`, `AceiteBasesLegais` |
| CEP / matrícula (AdministraCeps) | `AdministraCep`: transferência de CEPs entre matrículas (NV: significado de "matrícula") |
| Widgets / dashboard | `Widget`, `WidgetColaborador`, `Home/Widgets`; `DashboardSmarkBI` é módulo contratado |
| Assistente de relatório | Relatórios montados pelo usuário: `RelatorioModelo*`, `RelatorioFormula*` |
| Clientes com customização no código | Vivante, Gocil (`BIGocil`), Renner (`ConsoleWS_Renner`, API `Renner`), Sodexo (`ImportarPathSodexo`, comentário de CSP), Unimed/CBS (`UnimedCBS`), Bauer (guia antigo) |

## Plataforma
| Termo | Significado |
|---|---|
| Empresa / tenant / cliente | Linha em `Empresas` (BACKOFFICE) com seu banco de dados próprio; `subdominio` identifica a empresa |
| EMPRESAS / BACKOFFICE | Prefixos de script: banco do tenant x banco central |
| Instância (`LoginInstanciaDTO`) | Usuário no banco do tenant (`Logins`), com `idLogin` do tenant; `idLogin` da sessão (`LoginSessionDTO`) é o do BackOffice |
| Grupo / admin / admin CRM | `Grupo` com flags `admin` e `adminCrm`; permissões por menu e especiais |
| Permissão especial | `TipoPermissaoEspecial` (ex.: PodeEncerrarNegociacoes 1, ImpedirQueUsuarioRetrocedaEtapasDoFunilDasNegociacoes 3) |
| Módulo | `ModulosDoSistema`: o que a empresa contratou |
| Menu | Linha em `Menus` (`path`, `enumMenu`); define acesso às telas |
| Configuração | `NomeConfiguracao` + tabela `Configuracoes` (por empresa) |
| Parameter Store / SSM | AWS Systems Manager Parameter Store: configuração da plataforma (chave `AWSParameterStoreEnvironment`) |
| SMSSO | Cookie do SSO compartilhado com o Smark.UI; payload em Redis |
| JobHost | Serviço Windows `JobHostServices` (fila, trial, BI) |
| Queue / QueueFile / QueueMessage / QueueProcess | Fila: arquivo/resultado no tenant, mensagem e processo no BACKOFFICE |
| Trial | Empresa de teste criada/removida por processos do JobHost e pela API de trial |
| Mark / MyBI | Integrações de IA/BI do produto (`MarkAPI`, `GetUsuarioSmarkMyBI`, `AutenticarAPIMark`) (NV) |
| Smark.* | Projetos net9/net10 da migração; `SiteExpress.SMark.*` = legado em produção |

## Siglas
BLL (regras), DAL (dados), DTO, VM (ViewModel), SearchVM (filtro de pesquisa), EF6, SSO, SSM, HLG (homologação), PRD (produção), WI (work item do Azure DevOps), TFS (controle de versão antigo; guias em `doc/`), LGPD, CAEPF, CNAE, CID, IBGE (municípios), RE (`IntegracaoRE`, NV), ERP, BI, IA, CSP (cabeçalhos de segurança, `IsCspFilter`), SDK-style (csproj moderno) x csproj antigo (UI, `packages.config`).
