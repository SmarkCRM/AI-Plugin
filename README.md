# AI-Plugin

Plugin de skills e agentes para desenvolvimento no Smark, distribuído como marketplace do Claude Code.

## Conteúdo

| Plugin | Skill | Para quê |
|---|---|---|
| `smark-dev` | `work-item` | Atuar num work item do Azure DevOps (`crmsmark/SMark`): análise, branch, implementação, validação e entrega. Identifica o repositório (`smark-crm`, `smark-core`, `smark-backoffice`) e segue o perfil dele em `skills/work-item/reference/repos/`. |
| `smark-dev` | `run-crm-local` | Subir o SMark CRM em localhost: Redis, `SiteExpress.SMark.BackOffice.Api` e `SiteExpress.SMark.UI`. |
| `smark-dev` | `smark-crm-conhecimento` | Base de conhecimento verificada no código: SMark CRM legado 4.8, BackOffice novo e BackOffice MVC 4.8 (legado). Índice em `SKILL.md`, arquivos em `reference/`. |

### Agentes

| Plugin | Agente | Para quê |
|---|---|---|
| `smark-dev` | `smark-crm-especialista` | Especialista (somente leitura) no SMark CRM e no BackOffice novo: responde dúvidas e esclarece o planejamento de correção de bug e de melhoria, usando a base de conhecimento e conferindo no clone local. A skill `work-item` o consulta nas etapas 4 e 6 para `smark-crm` e `smark-backoffice`. |

Como acionar: peça pelo nome ("use o `smark-crm-especialista` para analisar este plano"), ou deixe o Claude delegar pela descrição do agente. Instalado, ele aparece como `smark-dev:smark-crm-especialista`.

## Instalação

No Claude Code (terminal ou aba Code do app):

```
/plugin marketplace add SmarkCRM/AI-Plugin
/plugin install smark-dev@smark
```

Para receber atualizações depois de um push neste repositório:

```
/plugin marketplace update smark
```

As skills do plugin aparecem com o prefixo do plugin (ex.: `smark-dev:work-item`) e ficam disponíveis em qualquer repositório, não só no `smark-crm`.

### Testar uma alteração antes do push

Aponte o marketplace para o clone local, em vez do GitHub:

```
/plugin marketplace add W:\Smark\Projetos\AI-Plugin
/plugin install smark-dev@smark
```

Depois de editar os arquivos, rode `/plugin marketplace update smark` e abra uma sessão nova para carregar a versão alterada. Para voltar ao GitHub, remova o marketplace local (`/plugin marketplace remove smark`) e adicione de novo com `SmarkCRM/AI-Plugin`.

### Skills duplicadas no projeto

As skills `work-item` e `run-crm-local` também existiam em `.claude/skills/` do repositório `smark-crm`. Com o plugin instalado, uma sessão nesse repositório mostra as duas versões (`work-item` e `smark-dev:work-item`). O plugin é a fonte oficial: depois de instalá-lo, apague as cópias do projeto e faça as alterações só aqui.

## Pré-requisitos da `work-item`

- PAT do Azure DevOps na variável de ambiente de usuário `AZURE_DEVOPS_EXT_PAT`, com os escopos Work Items (Read & write) e Code (Read & write). A skill explica como configurar e nunca pede o PAT no chat.
- Clones locais dos repositórios em que for atuar (ex.: `W:\Smark\Projetos\smark-git`, `apicore`, `backoffice`).

## Estrutura

```
.claude-plugin/marketplace.json        # catálogo do marketplace "smark"
plugins/smark-dev/
  .claude-plugin/plugin.json           # manifesto do plugin
  agents/smark-crm-especialista.md     # agente especialista (somente leitura)
  skills/work-item/                    # SKILL.md + reference/ (API e perfis por repo)
  skills/run-crm-local/
  skills/smark-crm-conhecimento/       # SKILL.md (índice) + reference/{crm,backoffice-novo,backoffice-mvc}/
```

## Manter a base de conhecimento

Cada arquivo de `skills/smark-crm-conhecimento/reference/` começa com o commit do repositório em que foi verificado. Depois de mudanças grandes no `smark-crm` ou no `smark-backoffice`, reverifique os arquivos afetados contra o código e atualize esse cabeçalho. Para saber o que mudou: `git log --oneline <commit do cabeçalho>..HEAD -- <caminho>`.

Regra da base: **nunca** colocar valores de segredo (chaves, senhas, connection strings). Os repositórios têm credenciais versionadas; a base só cita caminhos e nomes de chaves. Como este repositório vai para o GitHub, rode uma busca por `password`, `pwd=`, `AKIA`, `Server=` e `Data Source=` antes de cada push.

## Adicionar um repositório à `work-item`

Crie `plugins/smark-dev/skills/work-item/reference/repos/<repo>.md` seguindo o formato dos perfis existentes (Branch, Arquitetura e padrões, Configuração local, Scripts SQL, Build, Testes, Validação manual, Entrega) e adicione a linha na tabela "Repositórios e perfis" do `SKILL.md`. Suba a `version` em `plugin.json` a cada mudança publicada.
