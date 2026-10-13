# BackOffice novo — convivência com o legado (BackOffice 4.8, CRM, JobHost)

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: o que é compartilhado entre o BackOffice novo e o legado (4.8 MVC, BackOffice.Api, CRM web/API, JobHost), onde um bug ou mudança de um lado afeta o outro e o que de fato existe da "convivência via YARP". O legado em si é documentado por outros agentes.

## YARP: descrito, não implementado

README e ADRs 0001/0002/0005/0008 falam em reverse proxy (YARP) na frente das duas apps, roteando `/backoffice/<modulo>*` para a app nova e o resto para o 4.8. **Não há YARP em nenhum csproj, config ou código** deste repo, nem no repo legado `smark-git` (só menções em material de ferramentas `.claude/`, ignorado). Consequências verificadas:
- A app nova não tem rota base (`/backoffice/...`) nem proxy reverso; ela serve SPA e API na raiz da própria origem.
- Qual host/infra roteia por rota em hlg/prd: (não verificado) — pode existir fora dos repositórios.
- "Sem SSO": o cookie `smark.backoffice.session` é próprio; o 4.8 usa `FormsAuthentication`. O usuário loga nas duas apps separadamente.
- O README manda "roteie `/backoffice/<modulo>*` no YARP" ao migrar módulo: passo manual de infra, não rastreado no código.

## O que é realmente compartilhado

| Recurso | Compartilhamento | Impacto cruzado |
|---|---|---|
| Banco BACKOFFICE/CRM (`Logins`, `Empresas`, `Grupos`, `Menus`...) | mesmas tabelas, mesmas colunas | alterar semântica de coluna ou filtro (ex.: `Logins.deleted/ativo/Aplicacao`) afeta 4.8, CRM e JobHost; não há DDL em tabelas do CRM por regra dos ADRs |
| `Menus`/`MenusGrupo` | a app nova decide acesso pelo `path` do menu do 4.8 | renomear/mover menu no 4.8 ou em "Permissões" derruba o acesso na app nova; tela "Permissões" do novo grava `MenusGrupo` que o 4.8 também lê; o `Menus.Clear()` do 4.8 apaga vínculos (ADR 0007) |
| `Logins` (`replicar`) | a app nova marca `Replicar=true` ao gravar usuário de cliente; o **CRM** sincroniza (`RetrieveReplicacao/UpdateReplicacao`) | nome/e-mail só chegam ao CRM após a sincronização; não chamar replicação direta |
| Hash de senha | `SHA256(senha+salt)` igual ao CRM (`Auth:LegacyPasswordSalt`) | trocar o algoritmo/salt quebra login nos três lados |
| Bancos de empresa (tenant) | acesso direto por `Empresas.dataBase*`/`Servidores` | Avisos gravam `Notificacoes`/`LoginContador` que o CRM exibe (sino); conversão de trial replica logins e limpa dados no tenant; Fila altera `QueueFiles` |
| API do CRM (`SiteExpress.SMark.Api` → `CriaTrialController`/PostTrial) | HTTP em `CrmApi:BaseUrl` | criar trial envia e-mail de verdade; **nunca apontar dev/hlg para produção**; trial criado fora da app nova aparece como "Não registrado" em `TrialHistorico` |
| Queues API / Authentication API do CRM | HTTP (`QueuesApi:*`, `AuthenticationApi:*`) | e-mail de aviso entra na fila SQS consumida por worker fora destes repos; contrato `DadosAvisoEmailDTO` não pode mudar sem o worker |
| JobHost (`SiteExpress.SMark.JobHost`, .NET 4.8) | tabelas `QueueMessages`/`QueueProcess`; serviço Windows `JobHostServices` | app nova lista a fila e controla o serviço (SCM) — só Windows e com direito no SCM; job `RemoveEmpresaTrialProcess` apaga trial em `dtCriacao + 30 dias` (a prorrogação move `dtCriacao` por isso); JobHost legado deve gravar em `AuditoriaEventos` com os números de enum imutáveis, atrás da flag `AuditoriaEventosHabilitada` (outro repo; não verificado) |
| `BackOffice.Api` 4.8 (`Services/SiteExpress.SMark.BackOffice.Api`) | `JobHostController.RestartService` também reinicia o serviço | duas telas controlam o mesmo serviço; lock em memória da app nova não protege contra o 4.8; desligar exige confirmar que ninguém usa o endpoint antigo (ADR 0008, pendência) |
| MyBI / Route53 / S3 | integrações externas comuns | `MyBi:*` e `Route53:*` por ambiente; sem `MyBi:UrlBase` o novo usa dublê que falha explícito |

## Estado por módulo (migrado vs 4.8)

Migrados (existe tela/endpoint novo; a tela 4.8 correspondente ainda existe no repo legado — se foi desligada em prod: não verificado): Login, Home, Avisos, Empresas, Gerenciar Usuários, Trials, Rel. Engajamento, Rel. Uso Funcionalidades, Auditoria, Onboards, Lista JobHosts, Permissões, Tokens IA. Não migrados (continuam só no 4.8): Gerenciar Licenças, Licenças, Módulos de Sistema (grid), Funcionalidades (uso), Usuários (Cadastros), Configurações (senha), Gerador de Senha, ajuste de limite de Tokens IA. Detalhe em `visao-geral.md`.

## Menus e scripts (cruzamento 4.8 x novo)

- Tokens IA: script 01 cria menus novos `/ia/tokens*`; script 02 **apaga** os `../IA/*` do 4.8 (guarda cópia `BKP_TokensIa_*`) e só deve rodar com a app nova em produção aceitando os dois caminhos e com `@P3Confirmado=1`. Publicar o código **antes** do script 02, senão a policy fica sem path conhecido (403 geral).
- Logs de Auditoria: menu novo `/ferramentas/logs-auditoria`, vinculado só aos grupos admin; o 4.8 não o mostra nem grava nessa trilha ("alterações feitas no 4.8 não são registradas aqui").
- Menus existentes do 4.8 (`../Ferramentas/Trials`, `Avisos`, `Empresas`, `ListaJobhosts`, `GerenciarUsuarios`, `Auditoria`, `Onboards`, `RelatorioEngajamento`, `RelatorioUsoFuncionalidades`) são reusados sem script de dados. Menus de ação separados (criar/converter/prorrogar trial) **não existem**: as 4 policies de Trials usam o mesmo menu.
- Scripts legados ficam em `smark-git/src/BackOffice/UI/Sql/Scripts por Sprint/<versão>/`; os da app nova, em `backend/src/SmarkBackOffice.Api/Scripts/<Sprint>/`.

## Diferenças de comportamento intencionais (comunicar ao suporte/negócio)

Avisos: publicar/revogar/editar travados por status; publicador = usuário logado; mensagem sanitizada. Trials: policy em todas as rotas, RN-019/021 passam a bloquear (400), conversão repetida = 409, sem rename de banco. Usuários de clientes: saldo de licenças bloqueia licenciar, guarda do último admin por empresa, empresa interna fora do escopo. Fila JobHosts: controle do serviço só para admin. Permissões: todas as ações exigem admin. Empresas/Avisos: login sem grupo é recusado e telas exigem menu (paridade com `Index()` do 4.8, mas aplicadas também à API).

## Perguntas a fazer antes de qualquer mudança que cruze legado

1. A coluna/tabela é lida pelo 4.8, CRM ou JobHost? 2. O 4.8 ainda tem a tela/ação equivalente em produção? 3. Muda path de menu ou grupo? 4. Há worker/serviço (Queues, JobHost) consumindo o contrato? 5. O script precisa rodar antes ou depois do deploy? 6. O banco de tenant é tocado (credencial por empresa)? 7. Quem repassa a mudança de rota no proxy da infra?
