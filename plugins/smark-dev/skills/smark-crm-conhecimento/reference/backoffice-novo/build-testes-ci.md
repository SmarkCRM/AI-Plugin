# BackOffice novo — build, testes, docker-compose e CI/CD

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: comandos de build/teste, como os testes funcionam (e o que eles NÃO garantem), infra local e o pipeline em `.github/`.

## Comandos

| Ação | Comando |
|---|---|
| Restore/build backend | `cd backend && dotnet restore && dotnet build -c Release` (warnings = erro, `TreatWarningsAsErrors`) |
| Testes backend | `cd backend && dotnet test` |
| Rodar API | `cd backend && dotnet run --project src/SmarkBackOffice.Api` (5109) |
| Frontend dev/build/lint/test | `cd frontend && npm install`, depois `npm run dev`, `npm run build`, `npm run lint`, `npm test` (`build` = `tsc -b && vite build`; `lint` = oxlint; `test` = `vitest run`) |
| Codegen | `npm run codegen` (API rodando) / `npm run codegen:contract` (do `contracts/openapi.json`) — ver `contrato-e-codegen.md` |
| Publicar (deploy único) | `dotnet publish` do projeto Api: o target `BuildAndPublishSpa` roda `npm install` (se faltar `node_modules`) + `npm run build` e copia `frontend/dist` para `wwwroot`. Perfil `Publicar.pubxml` (FileSystem, framework-dependent) |
| Infra local | `docker compose up -d` sobe só SQL Server 2022 (porta 1433). Credencial `sa` está **em texto no `docker-compose.yml`** (ver armadilhas) |

## Testes do backend (xUnit)

- `tests/SmarkBackOffice.UnitTests` (referencia Domain/Application/Infrastructure): validadores, serviços com fakes, auditoria (snapshot, ações derivadas, allowlist de escritas fora do ChangeTracker), políticas de senha, etc.
- `tests/SmarkBackOffice.IntegrationTests` (referencia a Api): `WebApplicationFactory<Program>` via `SmarkWebApplicationFactory`.
  - Fixa por variável de ambiente `ParameterStore__Enabled=false`, connection string de LocalDB fictícia e `CrmApi__BaseUrl` fictício (o CRM real **nunca** é chamado: envia e-mail de verdade).
  - Troca por dublês: `IUsuarioRepository` (login `admin`), `IGrupoRepository`, `IAcessoMenuConsulta` (`FakeAcessoMenuConsulta`: libera tudo ou só os paths dados, conta consultas), `IPermissoesRepository`, Avisos (repositório, fila, CRM, leitura, S3), `ITokensIaRepository`.
  - `CoberturaDeAutorizacaoTests`: varre todos os endpoints `/api/*` e falha se algum não tem policy/role (exceto login/logout e a lista "SoLogin") e confere que OpenAPI não vaza fora de Development.
  - Testes por módulo: `*EndpointsTests` (200/401/403/validação), `TelasLiberadas*Tests`, `*MapeamentoTests`, `*RepositoryTests`.
- Números aproximados: ~1 100 métodos `[Fact]/[Theory]` no backend; ~315 casos no frontend.
- **Testes de repositório/SQL (~33 arquivos) usam SQL Server LocalDB** (`(localdb)\MSSQLLocalDB`), criam o schema a partir do modelo EF (`EnsureCreated`) e **só rodam no Windows**; sem LocalDB (Linux/CI) cada teste sai **sem falhar** (verde sem executar). Eles não validam contra o schema real do CRM; paridade com o legado depende de scripts `homologacao-paridade/Conferencia` rodados à mão. Alguns testes usam SQLite em memória.
- Frontend: Vitest em ambiente node (sem jsdom/testing-library); lógica pura e `renderToStaticMarkup`.
- Padrão para teste novo de endpoint: derivar de um `*EndpointsTests` existente, usar `FakeAcessoMenuConsulta.Liberando(...)` e dublê do repositório do módulo; adicionar caso 401 e 403.

## CI (`.github/workflows/ci.yml`)

Gatilhos: push em `main` e `pull_request`. Três jobs em `ubuntu-latest`:

1. **backend** (`working-directory: backend`): `actions/setup-dotnet` 10.0.x → `dotnet restore` → `dotnet build --no-restore -c Release` → `dotnet test --no-build -c Release`. Em Linux os testes LocalDB não executam.
2. **frontend**: Node 20, `npm ci` (cache em `frontend/package-lock.json`) → `npm run lint` → `npm run build`. **Não roda `npm test`** (315 testes do front ficam fora do CI).
3. **contract-in-sync**: sobe a API, baixa `/openapi/v1.json` para `contracts/openapi.json`, regera o cliente e falha se houver diff (ressalva sobre ambiente Production em `contrato-e-codegen.md`).

Não há job de deploy/release, segurança (SAST/dependências), cobertura publicada nem execução dos scripts SQL.

Sobre o pipeline real: o repositório é hospedado no Azure DevOps (`dev.azure.com/crmsmark`), os commits citam "Merged PR NNNN" e work items do ADO, e **não há `azure-pipelines.yml`** no repo; a pasta `.github/` pode não ser o pipeline efetivamente usado — (não verificado). O repo legado `smark-git` tem `azure-pipelines.yml` próprio.

Deploy em hlg/prd (processo, servidor, IIS/serviço, variáveis de ambiente): (não verificado).

## Como validar uma mudança antes do PR

1. `dotnet build -c Release` (warnings quebram) e `dotnet test` (no Windows com LocalDB para exercitar os repositórios).
2. `npm run lint && npm run build && npm test` no frontend.
3. Se a API mudou: regenerar `contracts/openapi.json` e `schema.d.ts` e commitar.
4. Se há script SQL: rodar MIGRATE e ROLLBACK em dev (idempotência) e registrar evidência na pasta `evidencias/`.
5. Conferir `CoberturaDeAutorizacaoTests` e, se houver nova escrita fora do ChangeTracker, a allowlist de auditoria.
