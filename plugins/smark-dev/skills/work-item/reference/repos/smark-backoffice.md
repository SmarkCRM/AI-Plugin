# Perfil: smark-backoffice (BackOffice novo, .NET 10 + React)

- Repositório Azure DevOps: `smark-backoffice`
- Clone local de referência: `W:\Smark\Projetos\backoffice` (o caminho pode variar por dev; identifique pelo remote)
- Stack: backend .NET 10 minimal APIs em `backend/` (`SmarkBackOffice.slnx`; camadas Domain / Application / Infrastructure / Api); frontend React 19 + TypeScript (Vite, TanStack Router/Query, openapi-fetch) em `frontend/`; contrato em `contracts/openapi.json`; ADRs em `docs/adr/`.
- Skills de apoio (existem também em `.claude/skills` do próprio clone): `smark-bo-requisitos`, `smark-bo-arquitetura`, `smark-bo-backend`, `smark-bo-banco-dados`, `smark-bo-frontend`, `smark-bo-devops`, `smark-bo-qa`. Veja também o `README.md` do repo ("Como migrar um módulo novo").

## Branch
- Base: **`main`**, atualizada (`git switch main`, `git pull --ff-only`) antes de criar a branch do work item. A `branchOriginal` registrada para a entrega é `main`.
- Atenção: o `defaultBranch` do repositório no Azure DevOps está como `feat/auth-login`. Não use o default do servidor; use a `main`.

## Arquitetura e padrões
- Backend: entidade em `Domain/<Modulo>/`, DTOs/casos de uso/interface de repositório em `Application/<Modulo>/`, EF Core sobre a tabela real em `Infrastructure`, `Endpoints/<Modulo>Endpoints.cs` registrado no `Program.cs`.
- Frontend: `frontend/src/features/<modulo>/`, rota em `src/router.tsx`. `frontend/src/api/generated/schema.d.ts` **não** é editado à mão: regenere com `npm run codegen` (API rodando) ou `npm run codegen:contract` (a partir de `contracts/openapi.json`).
- Mudou endpoint ou DTO: atualize `contracts/openapi.json` e regenere o cliente.

## Configuração local (nunca commitar)
- `ParameterStore:EnvironmentParameterName` e `ParameterStore:AwsProfile` em `backend/src/SmarkBackOffice.Api/appsettings.Development.json`, e qualquer `ConnectionStrings` local. São por dev.
- `.kyroon/doc-sync-index.json` costuma aparecer modificado no `git status`: não leve para o commit, a menos que o work item seja sobre isso.

## Scripts SQL
- O BackOffice lê as tabelas reais do BACKOFFICE/CRM. Se a mudança exigir alteração de schema, pare e pergunte ao usuário onde o script deve ficar (normalmente no `smark-crm`, ver perfil dele).

## Build
- Backend: `dotnet build backend/SmarkBackOffice.slnx -c Debug -v minimal`.
- Frontend (se houver alteração em `frontend/`): `npm run build` (inclui `tsc -b`) e `npm run lint` dentro de `frontend/`. Se `node_modules` não existir, peça autorização antes do `npm install`.

## Testes
- Backend: `dotnet test backend/SmarkBackOffice.slnx` (`UnitTests` e `IntegrationTests`).
- Frontend: `npm run test` (vitest) dentro de `frontend/`, quando houver alteração no front.
- Para revisão completa, use a skill `smark-bo-qa`.

## Validação manual
- API: `dotnet run --project backend/src/SmarkBackOffice.Api` (http://localhost:5109). Front: `npm run dev` em `frontend/` (http://localhost:5173). Pare os processos depois da validação.

## Entrega: PR com auto-complete
- PR da branch do work item para a `main`, com `workItemRefs` para o work item.
- Auto-complete ligado: `mergeStrategy = noFastForward`, `deleteSourceBranch = true`, `transitionWorkItems = false`, `autoCompleteSetBy` = dono do PAT.
- A `main` do `smark-backoffice` **não tem nenhuma política** (nem build). Com o auto-complete ligado, o PR é concluído assim que for criado. Por isso, o build e os testes locais da etapa 8 são a única barreira: não abra o PR sem eles passarem, e avise isso no resumo da entrega.
- URL do PR: `https://dev.azure.com/crmsmark/SMark/_git/smark-backoffice/pullrequest/<id>`.
