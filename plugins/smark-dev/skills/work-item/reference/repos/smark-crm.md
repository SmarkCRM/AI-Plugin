# Perfil: smark-crm (CRM legado .NET Framework 4.8)

- Repositório Azure DevOps: `smark-crm`
- Clone local de referência: `W:\Smark\Projetos\smark-git` (o caminho pode variar por dev; identifique pelo remote)
- Stack: .NET Framework 4.8, ASP.NET MVC 5 + Web API 2, namespace `SiteExpress.SMark.*`. Os projetos `Smark.*` (net10.0/net9.0) são da migração e só entram se o work item pedir (ver `CLAUDE.md`).
- Skills de apoio: `smark-crm-requisitos`, `smark-crm-arquitetura`, `smark-crm-backend`, `smark-crm-banco-dados`, `smark-crm-frontend`, `smark-crm-devops`, `smark-crm-qa`, `validar-views-razor`, `run-crm-local`.

## Branch
- Base: a branch atual no início do trabalho (`branchOriginal`), normalmente `main`. A branch do work item sai do estado atual dela.

## Arquitetura e padrões
- Camadas: UI MVC (Areas/*/Controllers herdando `BaseController`) → BLL → DAL → Domain/Infra; `ServicesProxy` → BackOffice.Api; JobHost.
- Siga a skill da camada correspondente para os detalhes de cada uma.

## Configuração local (nunca commitar)
- `AWSParameterStoreEnvironment` nos `Web.config`/`App.config` (`env-localPedro`, `env-localCassio`, `env-localRegis`). Essas alterações são esperadas no `git status`: não as descarte e não as leve para o commit.

## Scripts SQL
- Par MIGRATE/ROLLBACK em `src/BackOffice/UI/Sql/Scripts por Sprint/<versão>/`, onde `<versão>` vem de `System.IterationPath` (ex.: `SMark\Sprint 3.33` → `3.33`). Se a pasta não existir, crie-a (numeração a partir de `01`). Nunca use a pasta mais recente pela ordem dos nomes.
- Registre os dois arquivos como `<Content>` no `SiteExpress.SMark.BackOffice.UI.csproj` (regras completas na skill `smark-crm-banco-dados`, R19/R20).

## Build
- MSBuild do VS (`vswhere -latest -find "MSBuild\**\Bin\MSBuild.exe"`), **sem `/restore`**, com `/p:Configuration=Debug /m /nodeReuse:false /v:minimal`.
- Compile o projeto de mais alto nível afetado (ex.: `src/SMark/Web/SiteExpress.SMark.UI.csproj`), que compila as dependências junto.
- Se o VS estiver no meio de um build, espere terminar antes de rodar.
- Se os `bin` aparecerem vazios ou incompletos depois do build, investigue (timestamps, processos) e informe o usuário. Não presuma a causa.

## Validações adicionais
- Diff com `.cshtml` (criadas ou alteradas): rode a skill `validar-views-razor` depois do build. O MSBuild não compila as views (`MvcBuildViews=false`). Corrija até o script dar `OK` antes de pedir teste manual.
- JS de tela: mantenha o par `.js`/`.min.js` registrado no `bundleconfig.json`.

## Testes
- Projetos em `src/Tests` e `*.Tests`, com `vstest.console.exe` ou `dotnet test -f net48`, conforme o projeto.

## Validação manual
- Skill `run-crm-local` (Redis + BackOffice.Api + UI). Pare tudo depois da validação.

## Entrega: PR com auto-complete
- PR da branch do work item para a `branchOriginal`, com `workItemRefs` para o work item.
- Auto-complete ligado: `mergeStrategy = noFastForward`, `deleteSourceBranch = true`, `transitionWorkItems = false`, `autoCompleteSetBy` = dono do PAT.
- Políticas da `main`: build obrigatório (pipeline 18) e vínculo com work item (não bloqueante). Não há revisor obrigatório, então o PR entra assim que o build passar. Avise isso no resumo da entrega.
- URL do PR: `https://dev.azure.com/crmsmark/SMark/_git/smark-crm/pullrequest/<id>`.
