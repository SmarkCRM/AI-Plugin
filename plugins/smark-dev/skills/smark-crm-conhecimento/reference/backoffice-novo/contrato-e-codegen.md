# BackOffice novo — contrato OpenAPI e codegen

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: explicar como o contrato da API é gerado, versionado e consumido pelo frontend, e como evitar quebrar o CI de contrato.

## Peças

| Peça | Local | Observação |
|---|---|---|
| Gerador | `Microsoft.AspNetCore.OpenApi` (`AddOpenApi`/`MapOpenApi` em `Program.cs`) | OpenAPI 3.1.1, título "SmarkBackOffice.Api v1", versão 1.0.0 |
| Exposição | `GET /openapi/v1.json` e Scalar em `/scalar/v1` | **só em Development**; em produção não existe (teste `OpenApi_nao_fica_exposto_fora_de_desenvolvimento`) |
| Contrato versionado | `contracts/openapi.json` | 95 paths; atualizado manualmente/por CI |
| Cliente tipado | `frontend/src/api/generated/schema.d.ts` (~250 KB) | gerado por `openapi-typescript`; nunca editar |
| Cliente HTTP | `frontend/src/lib/api.ts` | `openapi-fetch` com `createClient<paths>` |

## Fluxo de mudança de API

1. Alterar endpoint/DTO no backend (`Api/Endpoints`, `Application/<Modulo>`). Tipos de retorno dos delegates (`Results<...>`, `Produces*`, `WithSummary/WithDescription`) determinam o que sai no contrato.
2. Subir a API em Development (`dotnet run --project backend/src/SmarkBackOffice.Api`, porta 5109).
3. Atualizar o contrato: `curl -s http://localhost:5109/openapi/v1.json -o contracts/openapi.json`.
4. Regenerar o cliente: `cd frontend && npm run codegen:contract` (lê `../contracts/openapi.json`) **ou** `npm run codegen` (lê direto a API em `:5109`, **só reescreve `schema.d.ts`, não o `contracts/openapi.json`**).
5. Ajustar o frontend aos tipos novos (`npm run build` roda `tsc -b` e quebra se o cliente divergir do uso).
6. Commitar juntos: código da API + `contracts/openapi.json` + `schema.d.ts` + front.

Atenção: o README raiz cita apenas `npm run codegen` e o texto de `contracts/README.md` é vago sobre o passo 3; para o CI de contrato passar, `contracts/openapi.json` **e** `schema.d.ts` precisam estar sincronizados com a API.

## CI de contrato (`.github/workflows/ci.yml`, job `contract-in-sync`)

Sobe a API (`dotnet run --no-launch-profile --urls http://localhost:5109`), espera `/health`, baixa `/openapi/v1.json` para `contracts/openapi.json`, roda `npm ci` + `npm run codegen:contract` e falha com `git diff --exit-code` se `src/api/generated` ou `contracts/openapi.json` mudaram ("Contrato/cliente desatualizados"). Ressalva (não verificado em execução): como `--no-launch-profile` não define `ASPNETCORE_ENVIRONMENT`, a API pode subir em Production, onde o OpenAPI **não** é mapeado; nesse caso o `curl` do job falharia. Confirmar com quem mantém o pipeline. Também não está verificado se esse workflow é de fato executado: o remoto é Azure DevOps e não há `azure-pipelines.yml` neste repo.

## Armadilhas do contrato

- Mudar nome/tipo de propriedade de DTO ou de enum quebra `schema.d.ts` e as telas silenciosamente até o `tsc -b`; sempre buildar o front.
- Conferir no `contracts/openapi.json` como cada enum foi serializado (número ou string) antes de usá-lo no front.
- Endpoints que devolvem erros por `TypedResults.Problem`/`ValidationProblem` só aparecem no contrato com `ProducesProblem`/`Results<...>`; sem isso o front não tem tipo para o erro (por isso ele lê o corpo com `mensagemProblema`).
- `/health` aparece no contrato (tag `Infra`).
- TypeScript fixado em ~5.9 porque `openapi-typescript` quebra com 6.0 (ADR 0001).
