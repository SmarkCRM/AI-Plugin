---
name: work-item
description: Atua como agente de desenvolvimento num work item do Azure DevOps (crmsmark/SMark), em qualquer repositório com perfil (smark-crm, smark-core/apicore, smark-backoffice): identifica o repositório e adapta branch, build, testes e entrega ao perfil dele; lê e analisa a task, classifica como fix ou feat, esclarece dúvidas, planeja, cria a branch, implementa no escopo, valida build e testes e prepara a entrega (PR ou merge, conforme o repo) vinculada ao work item, sempre com aprovação explícita. Use quando pedirem para iniciar ou atuar num work item/task/bug/feature (ex.: "pega a task 13827", "atua no work item 12345"), analisar requisitos antes de codar, criar branch de trabalho, preparar commit ou abrir PR.
---

# Atuar em um work item do Azure DevOps

## Objetivo
Analisar, implementar e preparar a entrega de um work item no Azure DevOps com rastreabilidade, segurança e o mínimo de impacto fora do escopo.

## Entrada esperada
- Número do work item (ex.: `12345`). Se não vier, pergunte.
- Contexto adicional opcional dado pelo usuário.

## Princípios e limites obrigatórios
1. Não inventar informações que faltam no work item.
2. Não implementar com dúvida crítica em aberto. Se não conseguir montar um plano, peça instruções ao usuário sobre como atuar (ver etapa 6).
3. Não descartar alterações locais sem autorização explícita.
4. Nunca executar ações destrutivas (`reset --hard`, `clean` destrutivo, `push --force`).
5. Nunca alterar a `branchOriginal` diretamente como parte da implementação. A única exceção é o merge de entrega, quando o perfil do repo prevê merge direto e o usuário aprova.
6. Nunca commitar, fazer push, publicar comentário ou abrir PR sem aprovação explícita. Cada aprovação vale só para aquela ação.
7. Não ampliar o escopo silenciosamente.
8. Não declarar build ou teste como executado sem evidência (saída do comando).
9. Toda entrega (PR ou commit de merge) precisa estar vinculada ao work item. Sem vínculo, não prosseguir.
10. Não commitar mudanças de ambiente local. Cada perfil de repo lista quais são (ex.: `AWSParameterStoreEnvironment`).
11. As regras específicas de um repositório vêm do perfil dele. Nunca aplique o perfil de um repo em outro, e não atue em repo sem perfil sem instruções do usuário.

## Repositórios e perfis
Organização `https://dev.azure.com/crmsmark`, projeto `SMark`. Cada repositório tem um perfil com o que muda de um para outro (stack, branch base, configuração local, build, testes, validação manual e modo de entrega):

| Repositório Azure DevOps | Perfil | Entrega |
|---|---|---|
| `smark-crm` | [reference/repos/smark-crm.md](reference/repos/smark-crm.md) | PR com auto-complete |
| `smark-core` (apicore) | [reference/repos/smark-core.md](reference/repos/smark-core.md) | merge direto na `hlg`, sem PR |
| `smark-backoffice` | [reference/repos/smark-backoffice.md](reference/repos/smark-backoffice.md) | PR com auto-complete |

Os comandos da API REST estão em [reference/azure-devops-api.md](reference/azure-devops-api.md). Os do Git são feitos no clone local de cada repo (`git -C <clone> ...`).

Repo sem perfil (ex.: `smark-crm-migration`, `smark-ses`): pare e peça instruções ao usuário. Se ele quiser, proponha um perfil novo em `reference/repos/` seguindo o formato dos existentes.

## Acesso ao Azure DevOps

**Pré-requisito, antes da Fase 1:** autenticar com o **PAT (Personal Access Token)** do desenvolvedor, guardado na variável de ambiente de usuário `AZURE_DEVOPS_EXT_PAT`.
- Leia a variável do registro do usuário, porque ela pode ter sido criada depois que a sessão abriu: `[Environment]::GetEnvironmentVariable('AZURE_DEVOPS_EXT_PAT','User')`. Teste o acesso com uma leitura simples (ver referência).
- Se a variável não existir, estiver vazia ou der 401/203, pare e peça ao usuário para configurá-la **no terminal dele**, com o comando abaixo (o PAT precisa dos escopos Work Items: Read & write e Code: Read & write):
  ```powershell
  $s = Read-Host 'Cole seu PAT do Azure DevOps' -AsSecureString; [Environment]::SetEnvironmentVariable('AZURE_DEVOPS_EXT_PAT', [Net.NetworkCredential]::new('', $s).Password, 'User')
  ```
- **Nunca peça ao usuário que cole o PAT no chat.** Se ele colar mesmo assim, não use esse valor. Explique que o token ficou exposto no histórico, recomende revogá-lo e gerar outro, e peça que configure a variável como acima.
- Nunca imprima, grave em arquivo ou coloque em URL o valor do PAT. Use-o só dentro do comando, lido da variável.
- Não use `az boards` nem `az repos`: a extensão `azure-devops` 1.0.6 quebra com o `keyring` do Azure CLI 2.88 (`module 'keyring' has no attribute 'core'`). Use a API REST.

## Comunicação de progresso
Informe o progresso em blocos curtos com prefixo, sem monólogo interno:
- `[ANÁLISE]` etapa atual, fatos observados, decisões e justificativa resumida
- `[CÓDIGO]` arquivos e componentes identificados
- `[IMPLEMENTAÇÃO]` ações concluídas
- `[VALIDAÇÃO]` build, testes e critérios de aceite
- Informe bloqueios e a próxima ação sempre que houver.

---

## Fase 1: Descoberta e entendimento

### 1. Ler o work item por completo
Busque o work item com `$expand=all` e colete, quando existir:
- título, tipo, estado, criador (`System.CreatedBy`) e responsável;
- descrição (`System.Description`), passos de reprodução (`Microsoft.VSTS.TCM.ReproSteps`, em bugs) e critérios de aceite (`Microsoft.VSTS.Common.AcceptanceCriteria`). Esses campos vêm em HTML: converta para texto antes de analisar;
- comentários;
- histórico relevante (updates);
- relações: work items pai, filhos e relacionados, links, anexos (`AttachedFile`), commits e PRs (`ArtifactLink`);
- work items relacionados e dependentes (leia o título, estado e descrição de cada um).

Anexos: liste nome e tamanho. Se algum for necessário para entender a demanda (print, planilha, documento), peça permissão antes de baixar para o scratchpad.

Se houver limitação de acesso, reporte objetivamente o que foi acessado, o que não foi e qual informação está faltando.

### 2. Identificar o(s) repositório(s) e carregar o(s) perfil(is)
Descubra em quais repositórios o work item vai mexer, a partir de:
- o repo da sessão atual (`git remote get-url origin`);
- commits, PRs e branches já vinculados ao work item ou aos relacionados (`ArtifactLink`, que traz o id do repositório);
- a descrição (ex.: "API de integrações" → `smark-core`; "BackOffice novo" → `smark-backoffice`; tela do CRM → `smark-crm`);
- busca no código, quando ainda houver dúvida.

O nome do repo é o último segmento do remote, depois de `/_git/`. O remote pode estar em `dev.azure.com/crmsmark/...`, `crmsmark@dev.azure.com/...` ou `crmsmark.visualstudio.com/...`: todos valem.

Para cada repo afetado:
1. leia o perfil dele em `reference/repos/`;
2. localize o clone local: primeiro o diretório atual; depois as pastas vizinhas do clone atual (ex.: `W:\Smark\Projetos\*`), conferindo o remote de cada uma. Se não achar, pergunte o caminho ao usuário. Não clone sem pedido.

Informe no bloco `[ANÁLISE]` os repos identificados, os perfis carregados e os clones locais. Se não der para saber o repo com segurança, trate como dúvida crítica.

### 3. Classificar a demanda
- `fix`: correção de comportamento incorreto, regressão, erro ou bug.
- `feat`: nova funcionalidade ou ampliação funcional.

Apresente a classificação com uma justificativa curta, baseada em evidências do work item. O tipo do item (Bug, Task, User Story) é um indício, mas não decide sozinho. Se a classificação for ambígua, trate como dúvida crítica.

### 4. Confirmar se as informações bastam
Verifique se há resposta clara para:
- o problema ou necessidade;
- o comportamento atual;
- o comportamento esperado;
- os critérios de conclusão;
- as áreas afetadas;
- os riscos, dependências e ambiguidades.

Antes de concluir que algo falta, procure no código. Se as informações bastarem, siga para o plano. Caso contrário, não implemente.

### 5. Esclarecer dúvidas críticas
Faça perguntas só quando forem necessárias:
- uma dúvida por pergunta;
- dê preferência a múltipla escolha, com opções objetivas;
- inclua "Outro" quando fizer sentido;
- informe o impacto da resposta na implementação;
- não pergunte o que já está no work item ou no código.

Use a ferramenta de perguntas (`AskUserQuestion`) quando disponível. Caso contrário, use o modelo:

```
##### Pergunta <n>
<dúvida objetiva>

A. ...
B. ...
C. ...
D. Outro

**Impacto da resposta:** <efeito prático no desenho da solução>
```

Se o usuário não puder esclarecer uma dúvida crítica:
1. prepare um comentário no work item pedindo o esclarecimento. Quem mencionar segue a tabela de "Pessoas a mencionar" abaixo;
2. mostre exatamente o texto que será publicado e quem será mencionado;
3. só publique com aprovação explícita. Depois disso, aguarde a resposta sem implementar.

#### Pessoas a mencionar nos comentários

| Situação | Quem mencionar |
|---|---|
| Dúvida de implementação de **nova feature** (`feat`) | **Fabio Beckenkamp** (`fbeckenkamp@gmail.com`, id `d7d0f6e9-d8af-4c07-bb27-1af062b673c5`) |
| Comentário de encerramento (etapa 12) | Os testers do SMark: **Bruno** (`bruno@smark.com.br`, id `fb896c6a-eac9-4f1e-9d2d-f0203c5fa628`) e **Kevin Duarte** (`kevin.duarte@smark.com.br`, id `03d0395a-5c69-456c-a355-4884d3a25a38`) |
| Dúvida em um `fix` | Pergunte ao usuário quem mencionar |

O Fabio só deve ser mencionado em dúvidas de implementação de novas features, **nunca** no encerramento. Se algum id não funcionar na menção, busque de novo nos membros dos times do projeto (ver referência).

---

## Fase 2: Planejamento e preparação

### 6. Apresentar o plano antes de alterar código
O plano deve conter:
- resumo do entendimento;
- classificação (`fix`/`feat`);
- comportamento atual;
- comportamento esperado;
- critérios de aceite identificados;
- repositórios afetados e, em cada um, os projetos e arquivos, seguindo a stack e a arquitetura do perfil;
- estratégia de implementação;
- validações necessárias;
- testes previstos;
- riscos conhecidos;
- dúvidas não bloqueantes, se houver.

Espere a concordância do usuário com o plano antes da etapa 7.

#### Se não for possível montar um plano
Às vezes os requisitos estão claros, mas mesmo assim não dá para montar um plano confiável. Por exemplo:
- não foi possível localizar no código a funcionalidade ou o fluxo afetado;
- há mais de uma abordagem técnica viável, com trade-offs relevantes;
- a mudança parece exigir alterações de arquitetura, banco de dados ou integrações externas;
- não está claro como validar ou testar;
- a causa de um bug não pôde ser identificada ou reproduzida.

Nesses casos, **não improvise uma solução nem comece a implementar**. Pare e peça instruções ao usuário sobre como atuar:
1. diga o que foi entendido e o que foi investigado (arquivos, buscas, hipóteses descartadas);
2. explique exatamente o que impede o plano;
3. faça perguntas objetivas, seguindo as regras da etapa 5. Quando houver abordagens alternativas, apresente cada uma com seu impacto e uma recomendação;
4. só retome o planejamento quando tiver as instruções.

Isso vale também para a Fase 3: se durante a implementação o caminho deixar de ser claro, pare e peça instruções.

### 7. Preparar o repositório e a branch
Faça isto em cada repo afetado, no clone local dele:
1. verifique o estado do repositório (`git status`);
2. identifique as alterações locais existentes. As de ambiente listadas no perfil são esperadas: não as descarte e não as leve para o commit;
3. defina a `branchOriginal` conforme a seção "Branch" do perfil. Ela pode ser a branch atual (`git branch --show-current`) ou uma branch fixa, atualizada com `git pull --ff-only` (ex.: `hlg` no `smark-core`, `main` no `smark-backoffice`);
4. analise a estrutura e os padrões do projeto;
5. localize os arquivos e componentes relacionados.

Se alterações locais atrapalharem a troca de branch ou a criação segura da branch, bloqueie e reporte. Não use stash, reset nem checkout de arquivos sem autorização.

Crie a branch a partir da `branchOriginal` (`git switch -c <nome>`). O padrão de nome é o mesmo em todos os repos:
- bug: `fix/<Descricao-breve>`
- feature: `feat/<Descricao-breve>`

Regras de nome da descrição:
- primeira letra maiúscula e o resto em minúsculas;
- palavras separadas por hífen;
- sem acentos, caracteres especiais ou espaços;
- curta.

Exemplos: `fix/Erro-baixar-anexos`, `feat/Template-email-notificacao`. Se o work item tocar mais de um repo, use o mesmo nome de branch em todos.

Depois de criar a branch, informe, por repo, a branch original, a nova branch e a classificação.

**Mudar o estado para "In Progress":** assim que a branch for criada e o desenvolvimento começar, altere o estado do work item no Azure DevOps para `In Progress` (ver referência). Não precisa pedir aprovação, porque é regra do processo e vale para todos os repos. Se já estiver `In Progress` ou em um estado mais adiantado, não mexa. Informe a mudança no bloco `[IMPLEMENTAÇÃO]`.

---

## Fase 3: Implementação controlada

### 8. Implementar dentro do escopo
- Faça o mínimo de alteração necessária, sem sair do escopo.
- Respeite a arquitetura e os padrões do repo (seção "Arquitetura e padrões" do perfil, naming e estilo do código ao redor). Quando o perfil indicar skills de apoio, use-as para os detalhes de cada camada.
- Preserve a compatibilidade.
- Não faça refatoração que não tenha relação com o work item.
- Não adicione dependências desnecessárias.
- Não inclua dados sensíveis (connection strings, tokens, senhas). As configurações vêm do AWS Parameter Store.
- Script SQL novo: siga a seção "Scripts SQL" do perfil. Se o perfil mandar perguntar, pergunte antes de criar.

Se o plano ficar inválido durante a implementação:
1. interrompa;
2. relate a descoberta;
3. explique o impacto;
4. proponha um ajuste no plano e espere a aprovação.

---

## Fase 4: Validação e entrega

### 9. Validar antes de sugerir commit
Em cada repo afetado:
1. Revise o diff (`git diff`) e confirme que não há mudança fora do escopo.
2. Compile você mesmo pela linha de comando, **sempre**, com os comandos da seção "Build" do perfil, inclusive com o Visual Studio aberto. É assim que você vê e corrige erros antes de dar a implementação por terminada. Não peça ao usuário para buildar no lugar.
   - Se der erro, corrija e rode o build de novo até passar. Reporte os warnings novos que vierem dos arquivos alterados.
3. Rode as "Validações adicionais" do perfil, quando houver (ex.: `validar-views-razor` para `.cshtml` no `smark-crm`, regenerar o cliente da API no `smark-backoffice`).
4. Execute os testes relacionados com os comandos da seção "Testes" do perfil. Crie testes novos quando fizer sentido e houver projeto de teste para a área.
5. Valide os critérios de aceite. Para validação manual, siga a seção "Validação manual" do perfil e pare o que tiver subido ao terminar.

Reporte objetivamente:
- build: sucesso ou falha;
- testes executados, aprovados e falhos;
- validações manuais;
- limitações;
- itens não validados e o motivo.

### 10. Pedir aprovação para o commit
Apresente:
- resumo das alterações;
- arquivos relevantes modificados, por repo;
- resultado do build;
- resultado dos testes;
- riscos possíveis;
- mensagem de commit sugerida (igual em todos os repos):
  - bug: `fix(#<numero>): <descricao breve>`
  - feature: `feat(#<numero>): <descricao breve>`

Adicione ao commit só os arquivos do escopo (`git add <arquivos>`, nunca `git add -A`). Só commite com aprovação explícita.

### 11. Entregar conforme o perfil
Depois da aprovação do commit, crie o commit aprovado e siga a seção "Entrega" do perfil de cada repo. Cada passo que publica algo (push, PR, merge) precisa de aprovação explícita. Há dois modos.

**PR com auto-complete** (`smark-crm`, `smark-backoffice`):
1. peça aprovação e faça push da branch (`git push -u origin <branch>`);
2. abra o PR da branch criada para a `branchOriginal` via API, no repositório do perfil, com `workItemRefs` apontando para o work item (ver referência);
3. confirme o vínculo do PR com o work item antes de concluir.

O PR deve ter:
- título objetivo (pode repetir a mensagem do commit);
- descrição breve;
- vínculo com o work item, obrigatoriamente.

Confirme que a source branch é a branch do work item e que a target é a `branchOriginal`. Nunca assuma outra target. Depois de criar o PR, envie a URL completa: `https://dev.azure.com/crmsmark/SMark/_git/<repo>/pullrequest/<id>`.

Todo PR criado por esta skill deve sair com auto-complete ligado (ver referência), com estas opções:
- `autoCompleteSetBy` = o usuário dono do PAT (id via `connectionData`);
- `mergeStrategy` = `noFastForward`;
- `deleteSourceBranch` = `true`;
- `transitionWorkItems` = `false`, porque o estado do work item não deve mudar ao completar.

Confirme que o auto-complete ficou ligado, relendo o PR (`autoCompleteSetBy` preenchido). No resumo da entrega, diga quando o PR vai entrar, de acordo com as políticas descritas no perfil (ex.: depois do build no `smark-crm`; na hora, no `smark-backoffice`).

**Merge direto, sem PR** (`smark-core`): siga o passo a passo do perfil (merge `--no-ff` na `branchOriginal` e push, cada um com aprovação) e confirme o vínculo do commit com o work item relendo as relações dele (ver referência).

### 12. Comentar no work item após a entrega (obrigatório)
**Sempre** que uma entrega for feita (PR criado ou merge enviado), prepare logo em seguida um comentário no work item. Esta etapa não é opcional e não pode ser esquecida. Se o work item tocou mais de um repo, faça um comentário só, cobrindo todas as entregas. O comentário deve ter:
- um breve resumo da correção ou implementação (o que mudou para o usuário);
- o link de cada entrega: o PR, ou o commit de merge quando não houver PR;
- as menções de acordo com a situação, conforme a tabela "Pessoas a mencionar" da etapa 5. Na entrega normal, são os testers do SMark, **Bruno** e **Kevin Duarte**. Nunca mencione o criador da task por padrão.

Mostre o texto exato e quem será mencionado, e publique com aprovação explícita. Depois de publicar, releia o comentário e confira as menções e os acentos (ver referência).

Além do `In Progress` da etapa 7, não altere o estado do work item, a menos que o usuário peça.

---

## Critérios de sucesso
- A demanda foi classificada com justificativa baseada em evidências.
- A implementação só começou depois de entendimento suficiente e plano aprovado.
- O código foi alterado apenas dentro do escopo.
- Build e testes foram reportados com evidência.
- Os repositórios afetados foram identificados e cada um seguiu o próprio perfil.
- Commit, push e entrega (PR ou merge) foram feitos com as validações e aprovações corretas, e a entrega está vinculada ao work item.
- A comunicação foi objetiva e rastreável durante todo o fluxo.
