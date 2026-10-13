# BackOffice novo — roteiros (checklists)

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: roteiros prontos para investigar bug, planejar melhoria, adicionar módulo, perguntas antes de planejar e riscos por camada. Remete aos demais arquivos desta pasta.

## 1. Investigar um bug

1. **Classificar a camada** pelo sintoma: tela em branco/erro de tipo (front); HTTP 401/403/404/409/422/500 (API); dado errado/ausente (repositório/SQL); só em hlg/prd (config, dados, menu, tenant).
2. **401**: sessão expirada (8 h sliding) ou login recusado (sem grupo conta como credencial inválida). **403**: policy — qual? (`autenticacao-e-autorizacao.md`); o login tem o menu (path completo) em algum grupo e `idEmpresa > 0`? admin exigido? **404/409/422**: regra do serviço do módulo (`Application/<Modulo>`).
3. **Localizar o fluxo**: tela `frontend/src/features/<modulo>/` → hook `use<Modulo>.ts` → rota em `Api/Endpoints/<Modulo>Endpoints.cs` → serviço `Application/<Modulo>/` → repositório `Infrastructure/Persistence/Repositories/` ou adaptador `Infrastructure/Tenant/`.
4. **Reproduzir**: backend local (Parameter Store/connection string local), `/scalar/v1` para chamar a rota, ou teste de endpoint com `SmarkWebApplicationFactory`. Conferir `X-Correlation-Id` da resposta nos logs e no evento de `AuditoriaEventos` (campo `CorrelationId`).
5. **Dados**: comparar com o 4.8/CRM (mesmas tabelas); usar os scripts `NN-Conferencia-SomenteLeitura.sql`/`homologacao-paridade.sql` do módulo em `Api/Scripts/`. Lembrar dos filtros globais de `Logins` e de datas Brasília x UTC.
6. **Tenant**: erro por empresa? (credencial/servidor/base inexistente); nunca copiar connection string/senha para ticket ou log.
7. **Auditoria**: a trilha (`/ferramentas/logs-auditoria`) mostra quem fez o quê (antes/depois, sem campos sensíveis); alterações do 4.8 não aparecem.
8. **Causa em contrato**: se o front deu erro após mudança de API, comparar `contracts/openapi.json` x `schema.d.ts` x uso; rodar `npm run build`.
9. **Teste de regressão**: adicionar caso no `*Tests` do módulo (unit para regra, integração para rota/policy); lembrar que testes SQL só rodam no Windows.

## 2. Planejar uma melhoria

1. Dizer em qual módulo cai (`modulos.md`) e se toca mais de um (ex.: Uso Funcionalidades reusa o repositório de Engajamento; `Usuario`/`Grupo` são compartilhados por login, Permissões, Tokens IA, Dashboard, Trials, Usuários de clientes).
2. Decidir se é **só front**, **front+API** (contrato muda → codegen), **banco** (script MIGRATE/ROLLBACK, entidade EF, impacto no 4.8/CRM) ou **tenant**.
3. Regra de acesso: menu existente, novo menu (script de dados) ou admin? Mudar `Politicas`/`TelasLiberadas`/`NAV_ITEMS` juntos.
4. Auditoria: a entidade já é auditada? Escrita fora do ChangeTracker precisa de evento explícito + allowlist.
5. Compatibilidade com o legado: o 4.8 ainda escreve/lê? worker/serviço consome o contrato? (`convivencia-com-legado.md`).
6. Config nova: seção em `appsettings.json` (sem segredo), `*Options` em Application, validar no startup; documentar de onde vem em hlg/prd.
7. Testes e critérios de aceite: casos 200/401/403/validação, paridade com o legado quando aplicável, filtros na URL no front.
8. Ordem de entrega: script SQL em dev/hlg → deploy da API (que aceita o estado antigo e o novo) → script destrutivo depois (padrão do script 02 de Tokens IA).
9. Estimar risco/rollback: ROLLBACK do script, flag/config para desligar (ex.: `Dashboard:Historico:Habilitado`, `FilaJobHosts:Servico:Habilitado`).

## 3. Adicionar um módulo novo (visão fim a fim)

1. **Domain** `Domain/<Modulo>/` (entidades, enums). 2. **Application** `Application/<Modulo>/` (DTOs, `I<Modulo>Repository`, serviço, validators FluentValidation, `Options`); registrar em `Application/DependencyInjection.cs`. 3. **Infrastructure**: `Persistence/Configurations/<Entidade>Configuration.cs` (`ToTable` legado), `DbSet` em `AppDbContext`, `Repositories/<Modulo>Repository.cs`, registro em `Infrastructure/DependencyInjection.cs`; tenant em `Tenant/` se preciso. 4. **Script SQL** se houver tabela nova (`banco-de-dados.md`). 5. **Api**: `Endpoints/<Modulo>Endpoints.cs` com `MapGroup("/api/<modulo>")` + `.RequireAuthorization(Politicas.X)`, `Map<Modulo>Endpoints()` em `Program.cs`. 6. **Autorização**: constante+lista em `AcessoMenu.cs`, nome em `Politicas.cs`, `AddPolicy` em `Program.cs`, id em `TelasLiberadas.cs`. 7. **Auditoria**: registrar em `RegistroEntidadesAuditadas` e/ou eventos via `IAuditoriaService`. 8. **Contrato**: API em Development → `contracts/openapi.json` → `npm run codegen:contract`. 9. **Frontend**: `features/<modulo>/`, rota em `router.tsx`, `NAV_ITEMS`, `TELAS_COM_FILTRO_SALVO` se aplicável (`frontend.md`). 10. **Testes**: `*EndpointsTests`, `TelasLiberadas*Tests`, unit do serviço, front. 11. **Infra**: se houver proxy por rota, avisar quem mantém (o repo não tem YARP). 12. Atualizar `Scripts/README.md` e, se for decisão relevante, um novo ADR (próximo número livre: atenção à numeração duplicada).

## 4. Perguntas antes de planejar

- Qual ambiente/banco o problema aparece (dev local, hlg, prd) e com qual login/grupo? Ele tem o menu e `idEmpresa > 0`?
- A tela equivalente do 4.8 ainda está em uso? Há diferença de comportamento intencional (ADR) que explique o relato?
- A mudança toca tabela compartilhada com CRM/JobHost/4.8 ou banco de tenant? Quem é dono do contrato (Queues, PostTrial, JobHost)?
- Precisa de DDL? Já houve exceção aprovada à regra "sem mudança de banco"? Quem executa o script e quando (antes/depois do deploy)?
- Datas: de qual fuso vem/vai o dado? Há exibição em `America/Sao_Paulo`?
- Auditoria/LGPD: dado pessoal na trilha ou em exportação? Segredo mascarado?
- Volume: tabela grande sem índice (`UsoFuncionalidade`, `Notificacoes`)? Há limite/paginação/422 definidos?
- Quem é o PO/stakeholder que valida regras marcadas "validar com o PO" nos ADRs (Avisos 0008/0009, Usuários de clientes 0013)?
- A infra (host, proxy, identidade do SCM) permite a solução? (hospedagem não documentada)

## 5. Riscos por camada

| Camada | Riscos típicos |
|---|---|
| Frontend | tipo gerado desatualizado; guard x policy divergentes (`telas` x policy); filtro salvo restaurando parâmetro inválido; datas sem fuso; CSV com injeção de fórmula; 403 tratado com mensagem genérica |
| API/Endpoints | rota nova sem policy (teste vermelho); contrato mudou sem regenerar; erro não tipado fora do OpenAPI; vazar `ex.Message`/connection string; autorização só por sessão |
| Application | regra duplicada entre serviço e validator; status/concorrência sem `WHERE` condicional; serviço novo sem registro de DI (falha só no startup); ordem de efeitos entre sistemas (trial) |
| Infrastructure/EF | alterar filtro global de `Logins`; mapeamento diferente do schema real (testes não pegam); `ExecuteUpdate/Sql` sem auditoria; trigger x EF (`HasTrigger`); N conexões a tenants sem limite |
| Banco/Scripts | MIGRATE não idempotente; sem ROLLBACK; script destrutivo antes do código; DDL em tabela do CRM; compat level do SQL Server |
| Auth/Menu | path de menu como substring solta; `idEmpresa=0`; revogação com cache; admin lido da claim em vez do banco |
| Integrações | apontar para produção a partir de dev (CRM PostTrial, Queues, Route53); timeout; token/credencial técnica ausente; MyBI desligado |
| Legado/convivência | 4.8 reescrevendo dados migrados; menus removidos antes da app nova aceitar o novo path; lock do serviço JobHost não vale contra o 4.8 |
| CI/Deploy | testes SQL não rodam no CI Linux; front sem `npm test` no CI; deploy único exige `npm run build` no publish |
