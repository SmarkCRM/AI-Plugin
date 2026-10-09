# AI-Plugin

Plugin de skills e agentes para desenvolvimento no Smark, distribuído como marketplace do Claude Code.

## Conteúdo

| Plugin | Skill | Para quê |
|---|---|---|
| `smark-dev` | `work-item` | Atuar num work item do Azure DevOps (`crmsmark/SMark`): análise, branch, implementação, validação e entrega. Identifica o repositório (`smark-crm`, `smark-core`, `smark-backoffice`) e segue o perfil dele em `skills/work-item/reference/repos/`. |
| `smark-dev` | `run-crm-local` | Subir o SMark CRM em localhost: Redis, `SiteExpress.SMark.BackOffice.Api` e `SiteExpress.SMark.UI`. |

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
  skills/work-item/                    # SKILL.md + reference/ (API e perfis por repo)
  skills/run-crm-local/
```

## Adicionar um repositório à `work-item`

Crie `plugins/smark-dev/skills/work-item/reference/repos/<repo>.md` seguindo o formato dos perfis existentes (Branch, Arquitetura e padrões, Configuração local, Scripts SQL, Build, Testes, Validação manual, Entrega) e adicione a linha na tabela "Repositórios e perfis" do `SKILL.md`. Suba a `version` em `plugin.json` a cada mudança publicada.
