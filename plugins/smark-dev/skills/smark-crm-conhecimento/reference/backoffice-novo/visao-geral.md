# BackOffice novo — visão geral

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: dar o mapa de alto nível do BackOffice novo (repo `smark-backoffice`): o que é, stack, estrutura, como rodar e em que ponto está a migração do BackOffice 4.8. Os demais arquivos desta pasta detalham cada assunto.

Fonte: código real (`backend/`, `frontend/`, `contracts/openapi.json`, `.github/`, scripts SQL) + README e ADRs só como apoio secundário (divergências em `armadilhas-e-divida-tecnica.md`).

## O que é

Reescrita do BackOffice administrativo da Smark (operadores internos da Smark, não clientes) numa stack nova, migrando módulo a módulo do BackOffice 4.8 (`smark-git/src/BackOffice`, ASP.NET MVC .NET Framework 4.8, EF6, Gentelella/jQuery). Opera sobre o **mesmo banco SQL Server do CRM/BACKOFFICE** (tabelas legadas) e nos bancos de cada empresa (tenant) quando necessário. Origem do repo: Azure DevOps (`dev.azure.com/crmsmark/SMark/_git/smark-backoffice`); commits citam work items (#NNNNN) e PRs "Merged PR".

## Stack

| Camada | Tecnologia |
|---|---|
| Backend | .NET 10 (`net10.0`), ASP.NET Core minimal APIs, EF Core 10 (SqlServer), FluentValidation 12, Scalar (docs), `Microsoft.AspNetCore.OpenApi` |
| Frontend | React 19 + TypeScript 5.9, Vite 8, TanStack Router (rotas em código) + TanStack Query, Mantine 9 (+ `@mantine/charts`/recharts), TipTap (editor de avisos), `openapi-fetch` + `openapi-typescript` |
| Integrações | AWS Parameter Store (SSM), S3, Route53; Queues API (SQS) e Authentication API do CRM; PostTrial da API do CRM; MyBI; serviço Windows `JobHostServices` (SCM) |
| Testes | xUnit (backend), Vitest (frontend) |
| Banco | SQL Server; **sem EF Migrations**: o schema evolui por scripts SQL manuais em `backend/src/SmarkBackOffice.Api/Scripts/` |

## Estrutura do repositório

```
smark-backoffice/
  backend/                     solução SmarkBackOffice.slnx (net10.0)
    src/SmarkBackOffice.Domain | .Application | .Infrastructure | .Api
    tests/SmarkBackOffice.UnitTests | .IntegrationTests
    Directory.Build.props      Nullable, ImplicitUsings, TreatWarningsAsErrors=true
  frontend/                    SPA (src/features/<modulo>/, src/router.tsx, src/lib/, src/api/generated/)
  contracts/openapi.json       contrato exportado da API (95 paths)
  docs/adr/                    ADRs 0001–0014 (numeração com colisões, ver armadilhas)
  docker-compose.yml           só um SQL Server 2022 de dev
  .github/workflows/ci.yml     pipeline de CI (ver build-testes-ci.md)
```

Fora desta base de conhecimento, por decisão do usuário: pastas de ferramentas/IA geradas no repo (`.claude/`, `documentation/`, `artifacts/` e similares).

## Deploy único (API serve o SPA)

`SmarkBackOffice.Api.csproj` tem o target `BuildAndPublishSpa`: no `dotnet publish`, roda `npm install`/`npm run build` e injeta `frontend/dist` em `wwwroot`. `Program.cs` usa `UseDefaultFiles/UseStaticFiles` e `MapFallbackToFile("index.html")`. Ou seja, produção = **um processo .NET servindo API (`/api/*`) e SPA na mesma origem**. Perfil de publicação `Properties/PublishProfiles/Publicar.pubxml` (FileSystem, framework-dependent). Local de hospedagem em hlg/prd: (não verificado).

## Como rodar localmente (resumo)

1. Pré-requisitos: .NET SDK 10, Node 20+.
2. Backend: `cd backend && dotnet run --project src/SmarkBackOffice.Api` → `http://localhost:5109` (Scalar em `/scalar/v1`, OpenAPI em `/openapi/v1.json`, health em `/health`). Em Development o OpenAPI/Scalar ficam ligados; em outros ambientes, não.
3. Banco: duas opções em `appsettings.Development.json`: (a) `ParameterStore:Enabled=true` + `EnvironmentParameterName` (parâmetro SSM `env-local<dev>`) + `AwsProfile` (ex.: `mfa`, sessão MFA expira); (b) `ParameterStore:Enabled=false` e `ConnectionStrings:Default`. Detalhes em `autenticacao-e-autorizacao.md` e `banco-de-dados.md`.
4. Frontend: `cd frontend && npm install && npm run dev` → `http://localhost:5173`; o Vite faz proxy de `/api` para `http://localhost:5109` (cookie de sessão atravessa o proxy).
5. Login: tabela `Logins` do CRM (`Aplicacao = 2`, hash `SHA256(senha+salt)` legado). Não há usuário embutido nem dados em memória: todos os módulos leem tabelas reais.
6. `Properties/launchSettings.json` tem perfis `http` (5109) e `https` (7109/5109) com `ASPNETCORE_ENVIRONMENT=Development`. `Properties/Publicar.json` é um duplicado desse arquivo, sem função aparente.

## Módulos que existem (resumo; detalhe em `modulos.md`)

Home/Dashboard, Avisos (+ acompanhamento e leitura), Empresas (+ módulos, Smark BI/MyBI, histórico), Usuários de clientes (Gerenciar Usuários), Trials (criar, converter, prorrogar), Relatório Engajamento, Relatório Uso de Funcionalidades, Auditoria CNPJ, Logs de Auditoria, Onboards, Fila JobHosts (+ controle do serviço), Usuários do BackOffice (Permissões), Tokens de IA (3 telas), login/sessão. 14 grupos de endpoints em `/api/*`.

## Status da migração

Mapeado por comparação entre controllers do 4.8 e o código novo:

- **Já migrado**: Login, Home, Avisos, Empresas (+Módulos de sistema por empresa), Gerenciar Usuários, Trials, Relatório Engajamento, Relatório Uso Funcionalidades, Auditoria (CNPJ), Onboards, Fila JobHosts (`ListaJobhosts`), Permissões, Tokens IA (3 controllers). Mais itens **novos sem equivalente no 4.8**: Logs de Auditoria, histórico mensal de indicadores, prorrogação de trial, leitura de avisos, duplicar aviso.
- **Sem equivalente no código novo (aparentemente não migrado, não verificado se está planejado)**: `GerenciarLicencas` (solicitações de licença), `Licencas` (visão de licenças), `ModulosSistema` (grid do cadastro de módulos), `Funcionalidades` (MostraUsoFuncionalidades), `Usuarios` (Cadastros), `Configuracoes` (alterar/redefinir senha, senha expirada), `GeradorSenha`. Há um script de homologação `Sprint 1.1/00009-CustomerLicenses/homologacao-paridade.sql` sem código correspondente na main (possivelmente em outra branch; não verificado).
- **Recursos do 4.8 ainda não portados dentro de telas migradas**: ajuste de limite de tokens IA (só leitura no novo), bloqueio por tentativas de login/expiração de senha/forçar troca (não aplicados no login novo), exportação XLSX dos logs (só CSV), SSO entre 4.8 e novo (não existe).
- Convivência com o 4.8: ver `convivencia-com-legado.md`. **O proxy YARP descrito no README/ADRs não existe no código** (nem neste repo nem no legado); a coexistência hoje é por banco compartilhado e rotas/menus.

## Convenções gerais

- Nomes de módulos, classes, tabelas novas e mensagens em **PT-BR** (valores de enum de auditoria em inglês de propósito).
- Um módulo = pasta homônima em Domain/Application/Infrastructure + `Endpoints/<Modulo>Endpoints.cs` + `frontend/src/features/<modulo>/`.
- Toda rota `/api/*` exige policy (teste de arquitetura `CoberturaDeAutorizacaoTests` falha se não).
- Decisões e restrições recorrentes nos ADRs: evitar DDL em tabelas do CRM/4.8; tabelas novas só com script MIGRATE/ROLLBACK idempotente; sem segredos no repositório.
