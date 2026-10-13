# BackOffice novo — armadilhas, dívida técnica e divergências doc x código

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: lista do que costuma dar errado, riscos latentes e onde README/ADRs não batem com o código (prevalece o código). Valores de segredos NÃO são reproduzidos; só caminhos.

## 1. Divergências README/ADRs x código

| # | Documento diz | Código mostra |
|---|---|---|
| 1 | README/ADR 0001/0002/0005/0008: YARP na frente das duas apps | Nenhum YARP (csproj, config, código) neste repo nem no `smark-git`; a app nova serve SPA+API na raiz (deploy único) |
| 2 | ADR 0002: flag `UseInMemoryData` para módulos de demo (Avisos) | Flag e repositórios em memória não existem mais; tudo lê tabelas reais (README já diz isso) |
| 3 | ADR 0002/README: `appsettings.Development.json` "não commitado" | Está **versionado** (com `ParameterStore:Enabled=true` e parâmetro/profile de um dev); README sugere `skip-worktree` |
| 4 | ADR 0005: policies em `AutorizacaoExtensions.AddPoliticasBackOffice`, "nunca inline no Program.cs" | Não existe `AutorizacaoExtensions`; todas as policies estão **inline** em `Program.cs` |
| 5 | ADRs 0004/0005 citam `feat/auth-login` e ADR 0003 (reservado) | ADR 0003 não existe; ADR 0013 declara a branch ignorada; `AcessoMenu*` já está na main |
| 6 | Status "proposto" em ADR 0007-relatorio-engajamento, 0011, 0012, 0013 | Os quatro estão implementados e em uso (endpoints, telas, testes) |
| 7 | ADR 0010 D-03: XLSX; D-06: `POST /api/logs-auditoria/eventos-cliente` | XLSX **recusado** (400, "só csv"); endpoint `eventos-cliente` **não existe** (fase 2 pendente) |
| 8 | ADR 0010: scripts em `Scripts/Sem sprint definida/00017-AuditLogs/` | Pasta é `Scripts/Sprint 1.1/00017-AuditLogs/` (reorganizada no commit `a622ffa`) |
| 9 | `Scripts/README.md`: 00009-CustomerLicenses, 00017, 00018, 00024 em "Sem sprint definida" | Estão em `Sprint 1.1` (índice desatualizado) |
| 10 | ADR 0011: endpoint `GET /api/relatorio-uso-funcionalidades/exportacao` | Não existe; exportação = `GET /empresas?todos=true` |
| 11 | ADR 0008-jobhost cita `JobHostHeartbeat` | Nenhuma referência a heartbeat no código; a tela usa `QueueMessages`/`QueueProcess` |
| 12 | Numeração de ADR | Duplicada: dois `0007`, dois `0008`, dois `0009`; `0003` ausente |
| 13 | README: "`npm run codegen` atualiza o contrato" | Só reescreve `schema.d.ts`; `contracts/openapi.json` exige `curl` (ou o CI) |
| 14 | ADRs citam `documentation/` e specs 000NN | Pasta ignorada nesta base; só o código vale |

## 2. Segredos e credenciais versionados (só caminhos)

- `docker-compose.yml`: senha do `sa` do SQL Server de dev em texto.
- `backend/src/SmarkBackOffice.Api/appsettings.json`: `ConnectionStrings:Default` com usuário/senha `sa` (banco local) e `Auth:LegacyPasswordSalt` (salt do hash de senha do CRM).
- `backend/src/SmarkBackOffice.Infrastructure/DependencyInjection.cs`: constante com o mesmo salt como fallback.
- `appsettings.Development.json`: nome de parâmetro SSM e profile AWS pessoais (não é segredo, mas é config individual versionada).
- Pastas `Scripts/**/evidencias/` guardam saídas de execução em dev (podem conter nomes de host/banco e ids); revisar antes de compartilhar.
- Tests usam valores fictícios (senhas de exemplo) — sem segredo real identificado.
- Recomendação não aplicada: mover para user-secrets/ambiente; salt legado só pelo Parameter Store.

## 3. Armadilhas de comportamento

1. **Testes de repositório "passam" sem rodar** em Linux/CI: ~33 arquivos dependem de LocalDB (Windows); sem ele saem verdes. Rode no Windows para exercitar SQL/EF.
2. **Frontend sem `npm test` no CI**; TypeScript/lint/build sim.
3. **CI de contrato** pode falhar por rodar a API sem `Development` (OpenAPI só existe em Development) — não verificado em execução; e o workflow em `.github/` talvez nem seja o pipeline real (repo no Azure DevOps).
4. **Menu = acesso**: se o menu do 4.8 for renomeado/removido ou o grupo não o tiver, a tela vira 403. Sempre path completo; path novo (`/ia/tokens*`, `/ferramentas/logs-auditoria`) é comparação exata, antigo é "contém". `idEmpresa > 0` no login é exigência embutida (login de BO sem empresa leva 403 em tudo com policy de menu).
5. **Policies de Trials iguais**: criar/converter/prorrogar usam o mesmo menu; não dá para permitir só uma ação (RB002/RB003 não atendidos) sem script de menus novos.
6. **Dashboard só com sessão** (por decisão): qualquer operador logado, mesmo sem grupo de menu, lê contadores. Lista explícita em `CoberturaDeAutorizacaoTests`.
7. **Admin em Uso de Funcionalidades**: a policy deixa admin passar sem menu, mas `/api/auth/me.telas` não inclui `relatorio-uso-funcionalidades` para ele (só engajamento tem a regra extra): o item de menu e o guard do front escondem a tela de admin sem o menu. (Lido no código; não executado.)
8. **Role `Admin` no cookie é cópia**: policies consultam o banco; mudar grupo admin vale na hora para a API, mas `/me.isAdmin`/menu do front só atualiza no próximo `/me` (cache 5 min) ou re-login.
9. **`/api/auth/me` faz ~17 consultas de menu (uma por tela) + checagens de admin** por chamada; o front cacheia 5 min.
10. **Login sem proteção de força bruta** e sem expiração de senha/forçar troca (existem colunas e flags, o login novo não aplica); só o 4.8 tem isso. Cookie com `SecurePolicy=SameAsRequest`: atrás de proxy TLS sem `Auditoria:ProxiesConfiaveis`, o esquema visto é http (cookie sem Secure) — não verificado em prod.
11. **Datas misturadas**: legado grava hora de Brasília sem fuso; `AuditoriaEventos` e vários DTOs novos usam UTC; `dataMarcadoLido` dos avisos sai "como gravada" (fuso do CRM não confirmado). Conferir módulo a módulo; `RelogioBrasilia` tem fallback UTC-3 fixo se o host não tiver fuso.
12. **Interceptor de auditoria**: falha ao gravar evento derruba a escrita EF (por design); `EnableRetryOnFailure` está desligado — ligar exige rever a transação do interceptor. Escrita nova fora do ChangeTracker sem evento+allowlist = lacuna silenciosa e teste vermelho.
13. **Alterações no 4.8 não entram na trilha** (mensagem fixa na tela de Logs).
14. **Fila JobHosts**: controle do serviço só funciona em Windows com `Habilitado=true` e direito no SCM; lock em memória por instância; Stop/Restart consomem tentativa dos itens em processamento; `Maquina` única por instância.
15. **Conversão de trial sem transação distribuída** (tenant → BACKOFFICE → Route53): falha de CNAME = 502 com conversão mantida; reenvio via UPSERT idempotente. Não renomeia o banco do tenant (diferença do `EmpresasController` 4.8).
16. **Criar trial** chama a API do CRM de verdade (envia e-mail): `CrmApi:BaseUrl` por ambiente, nunca produção em dev. "Quem criou" depende de extrair o subdomínio da URL devolvida pelo CRM (assunção do ADR 0014; confirmar em hlg), e trials criados por outros caminhos aparecem "Não registrado".
17. **Prorrogar** move `Empresas.dtCriacao` (+1..7 dias); trial já vencido por mais dias continua vencido.
18. **Avisos**: sem exclusão de rascunho nem status "Concluído"; publicar/revogar só por status; resultado do CRM por empresa não é persistido (some ao recarregar); consulta de leitura recusa (422) acima dos limites e depende de índice em `Notificacoes` (DBA); `Avisos.titulo` pode ter limite menor que o formulário (conferir coluna real).
19. **Usuários de clientes**: editar login com 2+ grupos remove o segundo se o grupo enviado difere (o ADR 0013 prevê aviso na UI; não verificado); "forçar troca de senha" escondido porque o CRM não respeita; nome/e-mail só chegam ao CRM após a replicação.
20. **MyBI**: sem `MyBi:UrlBase` usa dublê que falha explícito; `GerenciarUsuarioBanco` desligado em dev.
21. **`UsoFuncionalidade` sem índice** e relatórios agregando em memória: pode degradar com volume; nenhuma medição.
22. **Config só no startup**: `AddInfrastructure` lê opções ao subir; mudar `appsettings`/ambiente exige reiniciar.
23. **TreatWarningsAsErrors**: qualquer warning quebra o build/CI.
24. **Parameter Store só entrega a connection string**; as demais configurações sensíveis (AuthenticationApi, MyBI, CrmApi) vêm de onde? (não verificado).
25. Lixo/arquivos ocioso: `Properties/Publicar.json` (duplicado do launchSettings), `frontend/README.md` (boilerplate do template Vite), `@tanstack/router-plugin` instalado e não usado, `dotnet-tools.json` com `dotnet-ef` sem migrations.

## 4. Dívida e pendências registradas nos ADRs (continuam abertas no código)

- Hospedagem da API (SO, identidade com direito no SCM, nº de máquinas JobHost, uso externo de `JobHostController.RestartService`) — ADR 0008-jobhost, pendências 1–4 e 6.
- Portões de dados em hlg/prd (ADR 0006: grupos sem menu/sem grupo; ADR 0009-historico: `DataVirouEmpresa`, `dtCriacao`).
- Retenção de auditoria (ADR 0010 D-05, PO/jurídico) e XLSX (D-03); convergir `JobHostAcoesAuditoria` para a tabela central (D-08); auditar exportações feitas no browser (D-06).
- Menus de ação para separar permissões de Trials; rename de banco do tenant (depende do DBA); tabela/log de entrega por login do CRM (Avisos); reenvio de e-mails com erro.
- Aplicar bloqueio por tentativas, expiração de senha e `ForcarTrocaProximoLogin` (ADR 0002/0004) e definir SSO/ponte com o 4.8.
- Migrar o que falta do 4.8 (ver `visao-geral.md`) e ajuste de limite de Tokens IA.
