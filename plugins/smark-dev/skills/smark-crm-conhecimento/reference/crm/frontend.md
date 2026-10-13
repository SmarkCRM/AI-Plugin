> Verificado contra o código em 2026-10-13 (branch main, commit b13150a83).

# Frontend: Razor, JavaScript por tela, bundles e armadilhas de build

Objetivo: mostrar a estrutura de views/JS do CRM legado, as convenções de nome e as armadilhas de minificação, cache e compilação de views.

## Stack real
- Razor (MVC 5.2.8) + jQuery 3.7.1 + jQuery UI 1.14.1 + **Bootstrap 3.4.1** (tema core-admin `release_bs3`) + plugins jQuery: jqGrid (helper `Html.Grid` do MvcJqGrid), select2 **3.5.1**, bootstrap-datetimepicker, daterangepicker, meiomask, iCheck, TinyMCE, FullCalendar (2.2.3 e 3.10.2 na pasta), Highcharts 10.3.3, jKanban, OrgChart, DataTables.
- Não há npm/webpack para a UI. JS é escrito à mão; `.min.js` gerado por BundlerMinifier (extensão do Visual Studio) com `bundleconfig.json` (203 entradas) e `bundleconfig.json.bindings` (`produceoutput=true`, minifica ao salvar).
- Bootstrap 5 (`wwwroot/lib`) é do Smark.UI/migração; não usar nas telas legadas.

## Estrutura
- Layouts: `Views/Shared/_Layout_Pai.cshtml` (CSS/JS globais; define `urlContent`, `currentCulture`, `idEmpresa`, `login`, textos dos toasts; carrega `master.min.js`), `Views/Shared/_Layout.cshtml` (menu + cabeçalho), `Areas/<Area>/Views/_ViewStart.cshtml` (+ `Areas/Comercial/Views/Shared/_Layout*.cshtml`).
- View da tela: `Areas/<Area>/Views/<Controller>/Index.cshtml`; modais: `_Form.cshtml`, `_FormEdit.cshtml`, partials `_Algo.cshtml`. Partials de modal usam `Layout = null`.
- JS por tela: `Areas/<Area>/Scripts/<Controller>.js` + `.min.js`; objeto literal global `se<Controller>`; URL base `urlContent + '<Area>/<Controller>/'`. Núcleo: `Scripts/master.js` (`seBase`, `sePager`, `seConfiguracoes`, `seInitBootstrap`).
- Funções de `seBase` usadas em todo lugar: `renderForm(container, html, modal)` (injeta partial e inicializa plugins/validação unobtrusive), `executeScripJson(result)` (executa o `script` do `MensagemVM`), `mostrarMensagem('#divMsgSucess|Info|Warning|Error', texto)`, `reloadGrid`, `showLoading/hideLoading`, `setMascaras`. Todo `$.ajax` dispara o toast "Aguarde" (`$(document).ajaxStart`).
- Grids jqGrid: JS precisa expor `gv<Nome>Edit` e `gv<Nome>Delete` (os formatters chamam por nome).
- Textos: `Resources.Label.*` em `App_GlobalResources/Label.resx` (1.913 chaves) + `Label.en.resx` + `Label.designer.cs`; mensagens em `Mensagens.resx` (367 chaves). Acrescentar chave no `.resx` não atualiza o `Designer.cs` fora do Visual Studio; chaves de mensagem são lidas por string (`ResourceManager.GetString`), então funcionam sem designer.
- Cache-busting: `StaticFileCache.Version(path)` (`Helpers/StaticFileCache.cs`) acrescenta `?v=<dia><mês><ano><hora>.<ticks do arquivo>`: muda a cada hora **e** quando o arquivo muda.
- Variáveis: ViewBag de permissão `ViewBag.Adicionar/Editar/Deletar` (preenchidas em `BaseController.InicializarPermissoes`, ver armadilha em `backend.md`).

## Par `.js` / `.min.js` (fatos verificados)
- 164 `.js` em `Areas/*/Scripts`; só 6 não têm `.min.js` (`TarefaStatus`, `HorariosAgendamento`, `Unificacoes`, `Speech`, `html2pdfmake`, `jspdf.plugin.autotable`).
- **Não existe regra "a view sempre carrega o `.min.js`"**: as views fazem 319 referências a `.min.js` e 191 a `.js` sem minificar. Exemplo: `Comercial/Views/OportunidadesCards/Index.cshtml` carrega `OportunidadesCards.js` (não minificado) mas `AcoesDaOportunidade.min.js`; `BancoDeDados/Views/Templates/Index.cshtml` carrega `Templates.min.js`. Antes de editar, abra a view e veja qual arquivo ela carrega; mantenha os dois coerentes.
- Arquivo JS novo: entrada no `bundleconfig.json` (`outputFileName` + `inputFiles`) e, no `SiteExpress.SMark.UI.csproj`, `<Content Include>` do `.js` e do `.min.js` (este com `<DependentUpon>`). Sem isso não vai no publish.
- Fora do Visual Studio, `npx terser X.js -c -m -o X.min.js` funciona; `node --check X.js` valida sintaxe. Nunca editar `.min.js` à mão.

## Armadilhas de build/regeneração
1. **`Scripts/master.js` tem erro de sintaxe** (verificado com `node --check`: linha 337, `formaterAtivoGrid = function` dentro do literal de `seBase`; deveria ser `formaterAtivoGrid: function`). O `master.min.js` antigo funciona; regenerar a partir do `.js` pode quebrar ou mudar comportamento. Corrigir a sintaxe **antes** de minificar. Além disso, 14 views referenciam `~/Scripts/master.js` diretamente (ex.: `BancoDeDados/Views/Cargos/Index.cshtml`) e portanto carregam o arquivo inválido (não verificado o efeito no navegador).
2. **`applicationmin.min.css`** (`Content/themes/core-admin/release_bs3/build/stylesheets/`): o `.min.css` tem 20 usos de tokens `--azul` e 2 blocos `:root`; o fonte `applicationmin.css` tem 1 `:root` e **nenhum** `--azul`. Regenerar pelo `bundleconfig.json` apaga os tokens de cor. Estilos novos: `Content/Site.css` (+ `Site.min.css`) ou `@section styles`.
3. `MvcBuildViews=false`: erro em `.cshtml` (inclusive C# moderno não suportado no Razor/compilador do projeto) só aparece em runtime. Depois de mexer em view, abrir a tela de verdade.
4. UI é csproj antigo: view/JS/CSS novo precisa de `Content Include` (665 views listadas hoje).
5. Objeto de filtro duplicado: em `OportunidadesCards.js` o objeto `OportunidadeSearchVM` é montado em várias funções (lista, totais, exportação, kanban...). Campo novo: `grep` pelo campo vizinho e alterar todas as cópias; senão totais/kanban divergem da lista.
6. Eventos em HTML injetado: delegação (`$(document).on`) ou `off().on` para evitar bind duplicado. Plugins (select2, iCheck, datetimepicker via `data-mask`) já são inicializados por `seInitBootstrap`/`renderForm`; não reinicializar à mão.
7. URLs sempre com `urlContent + 'Area/Controller/...'` (o site pode rodar sob subdomínio/diretório virtual).

## Receitas
- Campo novo em modal de cadastro: `_Form.cshtml` (`LabelFor`/`TextBoxFor`/`ValidationMessageFor`, máscara por `data_mask`, select por classe `chzn-select`) + propriedade na VM/entidade + (se necessário) coluna no `Map` e script SQL.
- Coluna nova em grid: `colunas.Add(new Column(...))` na `Index.cshtml` e `cell = new[] {...}` na action `List` na **mesma ordem**.
- Período: input com classe `datepickerrange`, valor trafega como `DD/MM/YYYY - DD/MM/YYYY` e o parse é no controller.
- Formulário em modal: `Ajax.BeginForm` com `OnSuccess = "se<Tela>.afterSubmit"`, `OnBegin = "seBase.showLoading"`, `@Html.AntiForgeryToken()`. A chave `IsValidateAntiForgeryBySession=true` no `Web.config` indica validação de token por sessão (comportamento exato não verificado).
- Tela modelo de CRUD com lista + modal: `BancoDeDados/Templates`. Modelo de painel com filtros/cards: `Comercial/OportunidadesCards`.
- Testes de JS: `src/Tests/SiteExpress.SMark.Tests/Home/CalendarioAgendaTests.cs` carrega `.js` e `.min.js` num navegador headless (Playwright); há também `CalendarioAgenda.test.cjs` (node:test) possivelmente desatualizado (não verificado).
