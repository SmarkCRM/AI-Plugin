# BackOffice novo — banco de dados e scripts SQL

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: quais bancos/tabelas o BackOffice novo toca, quem é dono de cada uma e como funcionam os scripts SQL por sprint (único mecanismo de mudança de schema).

## Bancos

| Banco | Acesso | Uso |
|---|---|---|
| **BACKOFFICE / CRM central** (`SMarkBackOfficeContext` no Parameter Store; no dev local `SMarkBackOffice`) | EF Core (`AppDbContext`) via `ConnectionStrings:Default` ou Parameter Store | todas as tabelas abaixo; mesmo banco do CRM e do BackOffice 4.8 |
| **Banco de cada empresa (tenant)** | ADO.NET/SQL direto em `Infrastructure/Tenant/`; conexão montada de `Empresas.dataBase*` + `Servidores.nomeServidor` | Avisos (CRM), Fila JobHosts, conversão de trial, telefones, leitura de avisos, usuário MyBI |
| Dev local | `docker-compose.yml` sobe um SQL Server 2022 (`sa`, porta 1433) | só infra; **não há seed/scripts de schema automáticos** |

Não existe EF Migrations nem seed. O schema do BACKOFFICE é o do CRM/4.8; testes de persistência criam schema do modelo EF em LocalDB descartável (Windows) e **não** reproduzem o banco real.

## Tabelas mapeadas no `AppDbContext` (nome legado)

| Área | Tabelas |
|---|---|
| Login/permissões | `Logins`, `Grupos`, `GruposLogin` (join), `Menus`, `MenusGrupo` (join), `LoginHistoricoSenhas`; `LoginTentativaAcessos` (só UPDATE por SQL) |
| Empresas | `Empresas`, `Servidores`, `ModulosSistema`, `ModulosSistemaEmpresas`, `UsuariosSmarkBI`, `AbasMyBI` |
| Avisos | `Avisos`, `AvisoEmails`, `AvisoEmailEnvioLog` (leitura) |
| Tokens IA | `LogTokensAI`, `AuditoriaLimiteTokensAI` |
| Auditoria CNPJ | `AuditoriaConsultaCNPJ` |
| Onboards | `Onboards`, `OnboardsEmpresas` |
| Fila JobHosts | `QueueMessages`, `QueueProcess`, `JobHostAcoesAuditoria` (nova) |
| Relatórios | `UsoFuncionalidade`, `Funcionalidades`, `GrupoFuncionalidades` |
| Dashboard | `DashboardIndicadorHistorico` (nova) |
| Trials | `TrialHistorico` (nova) |
| Auditoria central | `AuditoriaEventos` (nova; trigger de imutabilidade + procedure de expurgo) |

No tenant (SQL direto, sem entidade): `Notificacoes`, `LoginContador`, `QueueFiles`, `Telefones`, `Logins` (replicação na conversão) e dezenas de tabelas limpas pelo `Tenant/Sql/LimparBaseTrial.sql` (Oportunidades, Contatos, Emails etc.). Qualquer mudança nelas afeta o CRM.

## Tabelas criadas/alteradas pelo BackOffice novo (resumo)

Novas (somente a app nova lê/grava; o 4.8 não conhece): `AuditoriaEventos`, `JobHostAcoesAuditoria`, `DashboardIndicadorHistorico`, `TrialHistorico`. Outros objetos: índices em `LogTokensAI(data)` e `AuditoriaConsultaCNPJ(DataRequisicao)`, colunas `Servico`/`IsCache` em `AuditoriaConsultaCNPJ`, `Avisos.mensagem` virou `NVARCHAR(MAX)` (emojis). Dados de menu: INSERT dos menus novos de Tokens IA (`/ia/tokens*`) e Logs de Auditoria (`/ferramentas/logs-auditoria`, liberado só a grupos admin); script que **remove** os menus antigos `../IA/*` do 4.8 (ver convivência).

## Scripts por sprint

Local: `backend/src/SmarkBackOffice.Api/Scripts/` (índice e regras em `Scripts/README.md`). **Não são aplicados pela app**: DBA/dev roda na ordem numérica da pasta, em dev e hlg antes de prd.

Convenção:

```
Scripts/<Sprint do backlog>/<NNNNN-NomeDaDemanda>/
  01-...-MIGRATE.sql / 01-...-ROLLBACK.sql     (par; MIGRATE idempotente)
  NN-Conferencia-SomenteLeitura.sql             (SELECT-only, rodar antes)
  90 - (LOCALDB) Simulacao-*.sql                (só banco local de simulação)
  evidencias/                                   (saídas das execuções; podem citar hosts/ids de dev)
```

- Sprint = `System.IterationPath` do work item no Azure DevOps (`Backoffice\Sprint 1.2` → `Sprint 1.2`). NNNNN = número da spec (pasta `documentation/` ignorada) ou id do work item.
- Prefixos nos nomes mais antigos: `(BACKOFFICE)`/`(LOCALDB)` indicam o banco-alvo.
- Pastas atuais (git, main):

| Sprint | Pasta | Conteúdo |
|---|---|---|
| Sprint 1 | `00003-AiTokenUsage` | menus Tokens IA (insert novos / delete legados, com trava `@P3Confirmado`), índice `LogTokensAI(data)` |
| Sprint 1 | `00005-PermissionsReview` | consultas de revisão/impacto de permissões, simulação local |
| Sprint 1 | `00006-EngagementReport` | homologação de paridade (consulta) |
| Sprint 1 | `00008-CnpjLookupAudit`, `00009-AuditScreenFix` | conferência, índice, colunas `Servico`/`IsCache` |
| Sprint 1 | `00010-OnboardsMigration` | conferência |
| Sprint 1 | `00015-DashboardHomeMigration` | conferência, criar `DashboardIndicadorHistorico`, verificação pós; README próprio |
| Sprint 1 | `00020-RelatorioUsoFuncionalidades` | homologação de paridade |
| Sprint 1.1 | `00009-CustomerLicenses`, `00017-AuditLogs`, `00018-CompaniesPageRefactor`, `00024-AvisosEmojis` | paridade de licenças (sem código na main), `AuditoriaEventos` + menu, conferência de Empresas, `Avisos.mensagem` nvarchar |
| Sprint 1.2 | `00011-JobHostsQueueMigration`, `10060-ProrrogacaoTrial` | `JobHostAcoesAuditoria`, `TrialHistorico` |

Última reorganização: commit `a622ffa` moveu 4 pastas de "Sem sprint definida" para `Sprint 1.1`; **`Scripts/README.md` ainda lista essas 4 como "Sem sprint definida"** e ADR 0010 aponta para esse caminho antigo (divergências).

### Para criar um script novo
1. Pasta `Scripts/<Sprint>/<NNNNN-Nome>/` (sprint do work item). 2. `NN-...-MIGRATE.sql` idempotente (`IF NOT EXISTS`, `SET XACT_ABORT ON`) + `-ROLLBACK.sql`. 3. Script de conferência somente leitura quando houver dado real envolvido. 4. Mapear a entidade/Configuration no EF (`ToTable`, nomes de coluna). 5. Atualizar `Scripts/README.md`. 6. Rodar primeiro em dev/hlg; nada é aplicado em prd pela app. Preferir não alterar tabelas do CRM/4.8 (restrição recorrente dos ADRs; exceções aprovadas: ADR 0014 `TrialHistorico`, ADR 0008 `JobHostAcoesAuditoria`, ADR 0009/0010).

## Pontos de atenção

- Compat level do banco de dev era 110 (< 130) mesmo em SQL Server 2022; `ISJSON`/`SESSION_CONTEXT` funcionaram. Conferir em hlg/prd (não verificado).
- `AuditoriaEventos` tem trigger: o mapeamento EF usa `HasTrigger` (senão o EF 10 usa `OUTPUT` sem `INTO` e falha).
- `Menus.enumMenu` NULL ao inserir menus novos: confirmar que a coluna aceita (pendência do ADR 0010).
- `Empresas.DataBasePassword` e senhas BI são segredos de tenant guardados no banco; nunca logar.
- Dados em hlg/prd dos "portões" dos ADRs (0006, 0008, 0009) **não verificados** neste levantamento: o código só foi conferido contra os scripts e ADRs.
- Tabelas sem índice conhecido: `UsoFuncionalidade` (relatórios) e `Notificacoes(idAuxiliar)` (leitura de avisos) — recomendações ao DBA fora do repo.
