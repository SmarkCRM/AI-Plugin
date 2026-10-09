# Perfil: smark-core (apicore, APIs .NET Core)

- Repositório Azure DevOps: `smark-core`
- Clone local de referência: `W:\Smark\Projetos\apicore` (o caminho pode variar por dev; identifique pelo remote)
- Stack: .NET Core (`netcoreapp3.1`), solution `SMark.NewArch.sln`, clean architecture. APIs em `src/SMark.*.Api` (Integrations, Webhooks, Mobile, Notifications, Authentication etc.), serviços em `src/SMark.*.Service`, domínio em `SMark.Domain`/`SMark.SharedKernel`, dados em `SMark.Infrastructure.Data`.

## Branch
- Base: **`hlg`**, sempre, independentemente da branch atual. Antes de criar a branch do work item:
  1. confira o `git status` (se houver alteração local que impeça a troca, bloqueie e reporte);
  2. `git switch hlg` e `git pull --ff-only` para atualizar;
  3. `git switch -c <nome>` a partir da `hlg` atualizada.
- A `branchOriginal` registrada para a entrega é `hlg`.

## Arquitetura e padrões
- Siga a organização da API afetada: endpoints/controllers na `*.Api`, regra de negócio na `*.Service`, entidades em `SMark.Domain`, acesso a dados em `SMark.Infrastructure.Data`.
- O mesmo work item pode tocar também o `smark-crm` (ex.: campo novo usado pelo CRM e pela API de integrações). Nesse caso, trate cada repo com o seu perfil.

## Configuração local (nunca commitar)
- `AWSParameterStoreEnvironment` (e `UsarParameterStore`) nos `appsettings.Development.json` das APIs. Alterações nesses valores são por dev: não as descarte e não as leve para o commit.

## Scripts SQL
- Se a mudança exigir alteração de schema, pare e pergunte ao usuário onde o script deve ficar antes de criá-lo.

## Build
- `dotnet build SMark.NewArch.sln -c Debug -v minimal` na raiz do clone. Para mudança restrita a uma API, pode compilar só o `.csproj` dela, mas confira se nada mais referencia o projeto alterado.
- O SDK 3.1 não está instalado; o build roda com o SDK mais novo e emite avisos de fim de suporte do `netcoreapp3.1`. Esses avisos são esperados; reporte só os warnings novos dos arquivos alterados.

## Testes
- Projetos em `tests/` (`SMark.Integrations.Api.Tests`, `SMark.Webhooks.Api.Tests`, `SMark.SharedKernel.Tests`, `SMark.Authentication.Api.Test`). Rode `dotnet test` no projeto de teste da área afetada.
- Se o teste não rodar por falta do runtime 3.1, reporte como limitação. Não instale runtimes sem pedido.

## Validação manual
- `dotnet run --project src/<Api>/<Api>.csproj` (portas: Integrations 8888, Webhooks 7777). Pare o processo depois da validação.

## Entrega: merge direto na `hlg`, sem PR
1. Commit na branch do work item com `feat(#<id>): ...` ou `fix(#<id>): ...` (com aprovação).
2. Com aprovação explícita do merge: `git switch hlg`, `git pull --ff-only`, `git merge --no-ff <branch> -m "Merge branch '<branch>' into hlg (#<id>)"`.
3. Com aprovação explícita do push: `git push origin hlg`. Faça push também da branch do work item só se o usuário pedir.
4. Confirme o vínculo relendo as relações do work item (`ArtifactLink` do commit). O Azure DevOps vincula pelo `#<id>` na mensagem. Se o vínculo não aparecer, reporte e pergunte como proceder.
5. Não abra PR, a menos que o usuário peça.
- Link para o comentário de encerramento: o commit de merge, `https://dev.azure.com/crmsmark/SMark/_git/smark-core/commit/<sha>`.
- A `hlg` não tem política de branch: o merge vale assim que o push for feito.
