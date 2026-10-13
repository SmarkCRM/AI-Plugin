> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Roteiros práticos: investigar bug, planejar melhoria, riscos por camada

Objetivo: checklists curtos para o especialista que responde dúvidas e valida planos de correção/melhoria no CRM legado (net48, `SiteExpress.SMark.*`). Detalhes de cada ponto estão nos outros arquivos desta pasta.

## 1. Investigar um bug no CRM
1. **Localizar a tela**: URL `/<Area>/<Controller>/<Action>` -> `src/SMark/Web/Areas/<Area>/Controllers/<Controller>Controller.cs`. "Painel de oportunidades" = `OportunidadesCards`. Se não houver Area, é `Web/Controllers`.
2. **Descobrir o que o navegador executa**: abrir `Views/<Controller>/Index.cshtml`, ver qual script carrega (`.js` ou `.min.js`) e a action chamada em `$.ajax`/`Ajax.BeginForm`. Conferir o objeto `se<Controller>` e se o filtro está montado em várias funções.
3. **Seguir a action**: permissão (`[CustomAuthorize]` e `AccessLevel`), conversão de strings (datas `dd/MM/yyyy - dd/MM/yyyy`), chamada à BLL. Lembrar do downgrade para Visualizar por nome de action e da casca de erro: `MensagemVM.Json`/chave de `Mensagens.resx`.
4. **Na BLL**: `BLL<X>` (`SiteExpress.SMark.BLL/<Pasta>`): `ValidarAdd/ValidarUpdate/ValidarDelete`, cópia VM -> DTO (campo faltando em algum método de pesquisa?), regras por `NomeConfiguracao`, `TipoPermissaoEspecial`, módulo contratado.
5. **Na DAL**: `DAL<X>`; para pesquisas de oportunidade, `montarfiltro`/`MontarFiltroFilho` e as CTEs; filtros implícitos (`deleted = 0`, `ativo`, tipo oportunidade x pós-venda, `NOLOCK`).
6. **Dados do tenant**: perguntar qual empresa/subdomínio. O banco muda por empresa; script `(EMPRESAS)` pode não ter rodado naquela base (coluna ausente, `Invalid column name`). Conferir `Configuracoes` e permissões/menus do grupo do usuário (banco BACKOFFICE).
7. **Sessão/SSO**: comportamento diferente entre logar no legado e vir pelo Smark.UI? Menus ficam na sessão (precisam novo login). Logout não limpa o SSO (ver `armadilhas-e-divida-tecnica.md`).
8. **Assíncrono**: exportação/importação/proposta? Ver `QueueFile` (tenant) e `QueueMessage` (BACKOFFICE): `status`/`statusExecucao`, `tentativas`, `messageErro`; log do JobHost (Serilog/Seq).
9. **Integração**: qual provedor; URL/credencial por cliente em `Configuracoes` ou chave no Parameter Store; endpoints anônimos de webhook sem sessão.
10. **Reproduzir**: subir Redis + BackOffice.Api (1999) + UI (62662) conforme `mapa-do-projeto.md`; para views, abrir a tela (o build não compila `.cshtml`).
11. **Teste**: só regras puras da BLL e JS têm teste automatizado (`src/Tests/SiteExpress.SMark.Tests`); o resto é validação manual.

## 2. Planejar uma melhoria/feature
1. Classificar a demanda (uma ou mais): filtro/campo em pesquisa existente; cadastro novo (tabela + CRUD); configuração por empresa; permissão especial; integração externa; processamento assíncrono; login/sessão/SSO (alto risco).
2. Achar um **precedente** no repo (tela irmã, `git log -S "<termo>"`) e copiar o padrão:
   - Cadastro novo: `Templates` -> `Domain/Models/Template.cs`, `Domain/Interfaces/Repositorios/BancoDeDados/IDALTemplates.cs`, `Infra/Mappings/TemplatesMap.cs`, `DAL/BancodeDados/DALTemplates.cs`, `BLL/Interfaces/BancodeDados/IBLLTemplates.cs` + `BLL/BancodeDados/BLLTemplates.cs`, `Web/Areas/BancoDeDados/Controllers/TemplatesController.cs`, `Views/Templates/{Index,_Form}.cshtml`, `Scripts/Templates.js` + `.min.js`, `bundleconfig.json`, `SiteExpress.SMark.UI.csproj`, `Label.resx` (+ `.en`), script `(EMPRESAS)` e `(BACKOFFICE) Insert-Menus` + `Menus` enum.
   - Filtro novo no painel de oportunidades: `Global.ServicesDTO/OportunidadeSearchVMDTO.cs`, `BLL/Models/OportunidadeSearchVM.cs`, `BLLOportunidades` (mapear em **todos** os métodos de pesquisa), `DALOportunidades.montarfiltro`, `OportunidadesCardsController.TratarDatasPesquisa` (se for período), `OportunidadesCards.js` (+ `.min.js` conforme a view carrega) e `Index.cshtml`.
   - Config por empresa: valor em `NomeConfiguracao` (`Domain/Enumeradores/Enumeradores.cs`), script `(EMPRESAS) Insert-Configuracoes`, tela em `Configuracoes/DeSistema`.
   - Fila: ver `filas-jobs-integracoes.md`.
3. Decidir **banco**: dado do cliente -> `(EMPRESAS)`; menu/permissão/fila/empresa -> `(BACKOFFICE)`. Pasta de script = sprint/patch do work item.
4. Decidir **configuração**: plataforma (Parameter Store) x tenant (`Configuracoes`). Nunca segredo novo em arquivo versionado.
5. Checar **efeitos colaterais**: oportunidade e pós-venda compartilham tabela e BLL; relatórios (`BLL/Relatorios`, `AssistenteRelatorio`) que usam o mesmo campo; exportação/fila; integrações que mapeiam o campo (`MapeamentoCamposIntegracao`, RD Station); campos obrigatórios por página (`PaginasComRegras`/`BLLValidacoes`); SQL de `vPesquisaOportunidade`/`vContasCard` em `src/BackOffice/UI/Sql/ScriptsBase` se o campo aparecer em cards.
6. Montar o plano por camada, com ordem: script SQL aprovado/aplicado -> Domain/Map -> DAL -> BLL (interface/registro Windsor só se injetada) -> Controller -> JS/View -> resources -> csproj/bundle. Marcar o que pode ir em paralelo (JS/View só precisam dos nomes de campos/actions).
7. Planejar validação: build net48 **e** net10.0 das libs (`dotnet build -f net48` e solution), abrir a tela no localhost, teste unitário se for regra pura. Commit `feat(#NNNNN): ...`/`fix(#NNNNN): ...`.

## 3. Perguntas que o especialista faz antes de planejar
- Qual work item, sprint/patch (`System.IterationPath`) e qual cliente/subdomínio reproduz?
- É oportunidade, pós-venda ou ambos? Painel (`...Cards`) ou cadastro? Lista, kanban, exportação?
- Quais perfis (grupo, admin, supervisor, permissão especial, módulo contratado) enxergam isso? Há configuração por empresa que liga/desliga?
- O campo/tabela já existe nas bases dos clientes? Em quantas bases o script vai rodar? Precisa de valor padrão para linhas existentes?
- Há integração (RD Station, Dinamize, DocuSign...) ou relatório/export que lê o mesmo dado?
- Impacta login, sessão, SSO ou `LoginSessionDTO`/`MenusSessionDTO` (afeta também o Smark.UI)?
- O texto da tela é novo (resources pt-BR e en)? Mensagem de erro de negócio nova (chave no `Mensagens.resx`)?
- O comportamento esperado vale só para o legado ou deve também entrar no Smark.UI?
- Qual é o critério de aceite observável na tela e qual perfil usar para testar?

## 4. Riscos por camada
| Camada | Riscos típicos |
|---|---|
| Controller/Authorize | Nome de action com `Get/List/Index/Pesquisar/Paginar` rebaixa permissão; tela sem menu fica negada; actions anônimas sem sessão; cookies de admin editáveis |
| BLL | Campo de VM/DTO não copiado em algum método; validação não executada (`Update` sem `validar`); chave de mensagem sem texto no `Mensagens.resx`; BLL sem `using` (contexto compartilhado) |
| DAL/SQL | Filtro só na lista (count/kanban divergem); SQL concatenado com texto livre; `NOLOCK` em leitura que decide escrita; `Delete` físico; coluna ausente em tenants sem o script |
| Banco/scripts | Pasta de sprint errada; `GO` em script; sem ROLLBACK/idempotência; script `(EMPRESAS)` pesado rodando em todas as bases; menu sem script BACKOFFICE |
| Domain/Infra | `Map` esquecido (coluna não lida); `AutoDetectChanges` off; mudança quebra net10.0 (Smark.UI) |
| Frontend | `.min.js` desatualizado ou view carregando o outro arquivo; `master.js` inválido; `.min.css` perde tokens; objeto de filtro duplicado; falta `Content Include`; erro de Razor só em runtime |
| Sessão/SSO/config | Mudança de payload quebra o Smark.UI; menus só após novo login; chave nova sem fallback no config base vira `null`; troca acidental de `AWSParameterStoreEnvironment` |
| Fila/JobHost | `QueueProcess.classe` diferente do nome da classe; tipo novo sem ramo no `Enqueue`; item travado em `Processando`; JobHost com flag desligada no ambiente |
| Integrações | Webhook sem autenticação; credencial de plataforma versus por cliente; chamadas síncronas (`.Result`) na requisição |

## 3. Precedentes reais (modelo de fatia completa)

Use `git log -S "<termo>" --oneline` ou o commit abaixo como molde antes de inventar um padrão. Cada linha foi conferida no histórico.

| Mudança | Commit | Arquivos tocados (resumo do que uma fatia vertical inclui) |
|---|---|---|
| Filtrar Segmentações por TAG (#13943) | `48a20ae1b` | `Global.ServicesDTO/TransferenciasCarteirasSearchVMDTO.cs` (campo novo) → `BLL/Models/TransferenciasCarteirasSearchVM.cs` (`tags` + `IdsTags` sanitizado) → `BLLContas.cs` (`ApplyTagFilter`), `BLLOportunidades.cs`, `BLLTagsEntidade.cs` → `DALOportunidades.cs` (`MontarFiltroTagsTransferencias`), `DALTagsEntidade.cs` → `SegmentacoesController.cs` → `Segmentacoes.js` + `.min.js` → `Views/Segmentacoes/_Form.cshtml` → testes em `src/Tests/SiteExpress.SMark.Tests/Comercial/` (`SegmentationTagIdsTests`, `TagFilterTests`). Sem script SQL. |

Observações do precedente:
- A tela **Segmentações** (Ferramentas) e **Transferências de Carteira** compartilham `TransferenciasCarteirasSearchVM`, o DTO e `PesquisarTransferenciasCarteirasDapper`: mudar o filtro afeta as duas.
- Contagem, listagem e "selecionar todos" (`ListarIdsTransferenciasCarteiras`) precisam receber o mesmo filtro, senão a lista mostra N e a execução em lote age em M.
- Não confundir com `SegmentacaoAcompanhamento` (lista de execuções) nem com `SegmentacoesSearchVM` (não parece ser o VM usado pela tela, não verificado).
- Os painéis Cards (`ContasCards`, `OportunidadesCards`, `PosVendasCards`) já filtram por tag com texto CSV (`DALContas`, `DALOportunidades`); a segmentação usa lista de ids sanitizada, que evita concatenar texto livre no SQL. Actions de tag: `BaseController.ListarTags` e `ListarTagsSelecionadas` (`Web/Controllers/BaseController.cs`).
