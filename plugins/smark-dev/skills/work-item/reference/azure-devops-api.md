# Azure DevOps: chamadas REST (PowerShell)

- Base: `https://dev.azure.com/crmsmark/SMark`
- Repositório: o do perfil (`smark-crm`, `smark-core`, `smark-backoffice`), em `$repo`

A autenticação é o PAT do desenvolvedor, na variável de ambiente de usuário `AZURE_DEVOPS_EXT_PAT`, via Basic auth. Nunca imprima o PAT.

```powershell
$pat = [Environment]::GetEnvironmentVariable('AZURE_DEVOPS_EXT_PAT','User')
if (-not $pat) { throw "AZURE_DEVOPS_EXT_PAT não configurada" }
$h = @{ Authorization = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$pat")) }
$base = "https://dev.azure.com/crmsmark/SMark"
$repo = "<repo do perfil>"   # ex.: smark-crm
```

Repita esse bloco no início de cada comando, porque o estado do shell não persiste entre chamadas.

Para testar o acesso sem expor nada:

```powershell
Invoke-RestMethod "https://dev.azure.com/crmsmark/_apis/projects?`$top=1&api-version=7.1" -Headers $h | Select-Object count
```

Um PAT inválido costuma devolver 401 ou 203, com uma página HTML de login no lugar do JSON.

## Leitura

```powershell
# Work item completo (campos + relações)
$wi = Invoke-RestMethod "$base/_apis/wit/workitems/<ID>?`$expand=all&api-version=7.1" -Headers $h
$wi.fields.'System.Title'; $wi.fields.'System.WorkItemType'; $wi.fields.'System.State'
$wi.fields.'System.CreatedBy'          # displayName, uniqueName, id (usar id na menção)
$wi.fields.'System.Description'        # HTML
$wi.fields.'Microsoft.VSTS.TCM.ReproSteps'               # HTML (Bug)
$wi.fields.'Microsoft.VSTS.Common.AcceptanceCriteria'    # HTML
$wi.relations | Select-Object rel, url, @{n='nome';e={$_.attributes.name}}
#   rel: System.LinkTypes.Hierarchy-Reverse (pai) / -Forward (filho) / Related / Dependency-*,
#        AttachedFile (anexo), ArtifactLink (commit/PR/branch), Hyperlink

# Comentários
(Invoke-RestMethod "$base/_apis/wit/workItems/<ID>/comments?api-version=7.1-preview.4" -Headers $h).comments |
  Select-Object createdDate, @{n='autor';e={$_.createdBy.displayName}}, text

# Histórico (updates)
(Invoke-RestMethod "$base/_apis/wit/workItems/<ID>/updates?api-version=7.1" -Headers $h).value

# Vários work items relacionados de uma vez
Invoke-RestMethod "$base/_apis/wit/workitems?ids=1,2,3&fields=System.Title,System.State,System.WorkItemType&api-version=7.1" -Headers $h

# Baixar anexo (só com permissão do usuário), usando a url da relação AttachedFile
Invoke-WebRequest "<url>?download=true&api-version=7.1" -Headers $h -OutFile "<scratchpad>\<nome>"
```

Para converter HTML em texto, basta algo simples:
`($html -replace '<br\s*/?>|</p>|</li>',"`n" -replace '<[^>]+>','') -replace '&nbsp;',' '`
Use `[System.Net.WebUtility]::HtmlDecode(...)` para decodificar entidades.

## Escrita (sempre com aprovação explícita)

```powershell
# Comentário com menção
$body = @{ text = "<div><a href=`"#`" data-vss-mention=`"version:2.0,<USER_ID>`">@<Nome></a> texto...</div>" } | ConvertTo-Json
Invoke-RestMethod "$base/_apis/wit/workItems/<ID>/comments?api-version=7.1-preview.4" -Method Post -Headers $h `
  -ContentType "application/json; charset=utf-8" -Body ([Text.Encoding]::UTF8.GetBytes($body))

# Alterar estado do work item (ex.: In Progress)
$patch = '[{"op":"add","path":"/fields/System.State","value":"In Progress"}]'
Invoke-RestMethod "$base/_apis/wit/workitems/<ID>?api-version=7.1" -Method Patch -Headers $h `
  -ContentType "application/json-patch+json" -Body $patch | ForEach-Object { $_.fields.'System.State' }

# Membros dos times do projeto (para achar o id de quem mencionar)
$teams = (Invoke-RestMethod "https://dev.azure.com/crmsmark/_apis/projects/SMark/teams?api-version=7.1" -Headers $h).value
$teams | ForEach-Object { (Invoke-RestMethod "https://dev.azure.com/crmsmark/_apis/projects/SMark/teams/$($_.id)/members?api-version=7.1" -Headers $h).value.identity } |
  Sort-Object id -Unique | Select-Object displayName, uniqueName, id

# Pull Request vinculado ao work item
$pr = @{
  sourceRefName = "refs/heads/<branch-do-work-item>"
  targetRefName = "refs/heads/<branchOriginal>"
  title         = "fix(#<ID>): <descricao>"
  description   = "<descricao breve>"
  workItemRefs  = @(@{ id = "<ID>" })
} | ConvertTo-Json -Depth 5
$r = Invoke-RestMethod "$base/_apis/git/repositories/$repo/pullrequests?api-version=7.1" -Method Post -Headers $h `
  -ContentType "application/json; charset=utf-8" -Body ([Text.Encoding]::UTF8.GetBytes($pr))
"https://dev.azure.com/crmsmark/SMark/_git/$repo/pullrequest/$($r.pullRequestId)"

# Ligar auto-complete no PR (o dono do PAT é quem "ativa")
$me = (Invoke-RestMethod "https://dev.azure.com/crmsmark/_apis/connectionData" -Headers $h).authenticatedUser.id
$ac = @{
  autoCompleteSetBy = @{ id = $me }
  completionOptions = @{ mergeStrategy = "noFastForward"; deleteSourceBranch = $true; transitionWorkItems = $false }
} | ConvertTo-Json -Depth 5
$r = Invoke-RestMethod "$base/_apis/git/repositories/$repo/pullrequests/<PR_ID>?api-version=7.1" -Method Patch -Headers $h `
  -ContentType "application/json; charset=utf-8" -Body ([Text.Encoding]::UTF8.GetBytes($ac))
"auto-complete por: $($r.autoCompleteSetBy.displayName)"   # vazio = não ligou

# Confirmar o vínculo PR -> work item
(Invoke-RestMethod "$base/_apis/git/repositories/$repo/pullRequests/$($r.pullRequestId)/workitems?api-version=7.1" -Headers $h).value
```

## Repositórios e vínculos de commit

```powershell
# Repositórios do projeto (nome <-> id)
(Invoke-RestMethod "$base/_apis/git/repositories?api-version=7.1" -Headers $h).value | Select-Object name, id

# ArtifactLink de commit/PR/branch no work item:
#   vstfs:///Git/Commit/<projectId>%2F<repoId>%2F<sha>
#   vstfs:///Git/PullRequestId/<projectId>%2F<repoId>%2F<prId>
#   vstfs:///Git/Ref/<projectId>%2F<repoId>%2FGB<branch>
# O <repoId> identifica o repositório (compare com a lista acima).
$wi.relations | Where-Object rel -eq 'ArtifactLink' | ForEach-Object {
  $p = [Uri]::UnescapeDataString(($_.url -split '/')[-1]) -split '/'
  [pscustomobject]@{ tipo = ($_.url -split '/')[-2]; repoId = $p[1]; ref = $p[2] }
}

# Entrega sem PR (merge direto): confirmar que o commit foi vinculado ao work item pelo "#<ID>" da mensagem
$wi = Invoke-RestMethod "$base/_apis/wit/workitems/<ID>?`$expand=relations&api-version=7.1" -Headers $h
$wi.relations | Where-Object { $_.rel -eq 'ArtifactLink' -and $_.url -like '*Git/Commit*' } | Select-Object url
# Link do commit para o comentário:
"https://dev.azure.com/crmsmark/SMark/_git/$repo/commit/<sha>"
```

Envie o corpo como bytes UTF-8, para não corromper acentos no PowerShell 5.1.

**Scripts `.ps1` com texto acentuado precisam ser salvos em UTF-8 _com BOM_.** O Windows PowerShell 5.1 lê `.ps1` sem BOM como ANSI (Windows-1252), e aí "integração" é publicado como "integraÃ§Ã£o" em comentários e PRs. Se o script for criado com a ferramenta Write, que grava sem BOM, regrave-o com BOM antes de executar:
`$f='<script>.ps1'; [IO.File]::WriteAllText($f, [IO.File]::ReadAllText($f, [Text.Encoding]::UTF8), (New-Object Text.UTF8Encoding $true))`.
Depois de publicar, releia o texto e confira os acentos. Ao procurar corrupção, use `-cmatch 'Ã'`: o `-match` não diferencia maiúsculas de minúsculas e casa também com `ã`.
