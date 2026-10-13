---
name: smark-crm-conhecimento
description: Base de conhecimento verificada no código do SMark CRM (legado .NET Framework 4.8, SiteExpress.SMark.*), do BackOffice novo (smark-backoffice, .NET + React) e do BackOffice MVC 4.8 (legado). Índice dos arquivos de referência sobre arquitetura, módulos, banco, sessão/SSO, filas, convenções e armadilhas. Use para responder dúvidas e planejar correção de bug ou melhoria nesses produtos; para uma análise completa, delegue ao agente smark-dev:smark-crm-especialista.
---

# Base de conhecimento SMark

Mapas curtos e verificados contra o código, em `reference/` (ao lado deste arquivo). Não substituem o código: cada arquivo começa com o commit em que foi verificado, e o que for decisivo deve ser reconfirmado no clone local antes de afirmar.

Para análise de bug, melhoria ou plano de work item, prefira o agente `smark-dev:smark-crm-especialista`, que usa esta base e confere no código.

## Qual produto?

| Produto | Repositório | Situação | Pasta |
|---|---|---|---|
| SMark CRM | `smark-crm` | Em produção; trabalho do dia a dia é no legado 4.8 | `reference/crm/` |
| BackOffice novo | `smark-backoffice` | Migrado, em evolução | `reference/backoffice-novo/` |
| BackOffice MVC 4.8 | `smark-crm` (`src/BackOffice`, `src/Services`) | Legado em descomissionamento; o CRM ainda depende da BackOffice.Api e de BLL/DAL compartilhados | `reference/backoffice-mvc/` |

## Onde procurar (CRM, `reference/crm/`)

| Pergunta | Arquivo |
|---|---|
| Quais projetos existem, camadas, portas, como rodar? | `mapa-do-projeto.md` |
| Qual Area/módulo cuida de X? Onde fica a tela? | `areas-e-modulos.md` |
| Como é Controller → BLL → DAL, IoC, validação, retorno AJAX? | `backend.md` |
| Banco do tenant (EMPRESAS) x central (BACKOFFICE), scripts, menus? | `banco-de-dados.md` |
| Views Razor, JS por tela, bundles, `.min.js`? | `frontend.md` |
| Autorização, permissões, SSO, Redis, configuração (sem valores)? | `seguranca-sessao-config.md` |
| Filas, JobHost, BackOffice.Api, integrações externas? | `filas-jobs-integracoes.md` |
| Pegadinhas, dívida técnica, suspeitas? | `armadilhas-e-divida-tecnica.md` |
| Como investigar um bug, planejar uma melhoria, riscos por camada? | `roteiros.md` |
| Termos de domínio e siglas? | `glossario.md` |
| Projetos `Smark.*` (net9/net10), regra de mexer no legado? | `migracao.md` |

## Onde procurar (BackOffice novo, `reference/backoffice-novo/`)

| Pergunta | Arquivo |
|---|---|
| O que é, stack, solução, como rodar, status da migração? | `visao-geral.md` |
| Que módulos/telas/endpoints existem? | `modulos.md` |
| Camadas, endpoints, DI, EF Core, auditoria, integrações? | `backend.md` |
| React, rotas, features, como criar uma tela? | `frontend.md` |
| OpenAPI e geração do client? | `contrato-e-codegen.md` |
| Tabelas legadas, scripts por sprint? | `banco-de-dados.md` |
| Login, autorização por menu, administrador? | `autenticacao-e-autorizacao.md` |
| Convivência com o CRM/4.8 e o que ainda falta migrar? | `convivencia-com-legado.md` |
| Build, testes, CI, docker-compose? | `build-testes-ci.md` |
| Divergências ADR x código, dívidas? | `armadilhas-e-divida-tecnica.md` |
| Investigar bug, planejar melhoria, adicionar módulo? | `roteiros.md` |

## BackOffice MVC 4.8 (legado, `reference/backoffice-mvc/`)

`visao-geral.md`, `modulos-e-telas.md`, `backend.md`, `banco-de-dados.md`, `backoffice-api-e-servicos.md` (BackOffice.Api, ServicesProxy, serviços), `seguranca-e-acesso.md`, `armadilhas-e-divida-tecnica.md`, `roteiros.md`. Use só para dúvidas do CRM que dependem desse código, ou quando o BackOffice novo ainda não cobrir o módulo (ver `backoffice-novo/modulos.md`).

## Clones locais (referência; o caminho varia por dev)

Identifique pelo `git remote -v` (último segmento depois de `/_git/`). Caminhos usados por quem montou esta base:

| Repositório | Caminho de referência |
|---|---|
| `smark-crm` | `W:\Smark\Projetos\smark-git` |
| `smark-backoffice` | `W:\Smark\Projetos\BackofficeKyroon\smark-backoffice` (o perfil da `work-item` cita `W:\Smark\Projetos\backoffice`) |

Se nenhum existir, peça o caminho ao usuário; não clone sem pedido.

## Como usar

1. Escolha o produto na primeira tabela e abra só os arquivos que a pergunta pede (cada um tem até ~12 KB).
2. Confirme no clone local o que for específico. Para saber se o código mudou depois da verificação: `git log --oneline <commit do cabeçalho>..HEAD -- <caminho>`.
3. Se achar divergência entre a referência e o código, vale o código; avise o time para corrigir o arquivo.

## Regras

- **Sem segredos.** Os repositórios têm credenciais versionadas; esta base só cita caminhos e nomes de chaves. Nunca copie valores para respostas, comentários de work item ou arquivos.
- A base cobre mapa e convenções, não regras de negócio detalhadas. Regra de negócio ausente aqui: leia o código ou pergunte ao time.
- Para atualizar a base depois de mudanças grandes, refaça a verificação do arquivo afetado contra o código e atualize o commit do cabeçalho.
