# BackOffice novo — frontend

> Verificado contra o código em 2026-10-13 (branch main, commit a622ffa).

Objetivo: mapa do SPA React (estrutura, rotas, guard de acesso, padrão de feature, filtros salvos) e receita para adicionar uma tela.

Raiz: `frontend/` — Vite 8, React 19, TypeScript ~5.9 (`strict`, `noUnusedLocals/Parameters`, `erasableSyntaxOnly`, `verbatimModuleSyntax`), oxlint, vitest. Gerenciador: npm (`package-lock.json`).

## Estrutura

```
frontend/src/
  main.tsx                    providers: Mantine (esquema auto), DatesProvider pt-br, QueryClient, ModalsProvider, Notifications, RouterProvider
  router.tsx                  TODAS as rotas (code-based, não geradas), guards e validateSearch
  lib/api.ts                  cliente openapi-fetch tipado (baseUrl '', credentials 'include')
  lib/queryClient.ts          staleTime 30s, retry 1, sem refetch no foco
  lib/erroHttp.ts             MSG_SEM_PERMISSAO, mensagemProblema(ProblemDetails), erroHttp(403 -> mensagem padrão)
  lib/theme.ts, index.css     tema Mantine + overrides
  api/generated/schema.d.ts   tipos gerados do OpenAPI (NÃO editar à mão)
  features/<modulo>/          telas, hooks, utilitários e testes do módulo
  features/shell/AppLayout.tsx  layout autenticado + NAV_ITEMS (menu lateral recolhível, estado em localStorage)
  features/auth/              LoginPage, auth.ts (query /api/auth/me), acesso.ts (guard), useLogin/useLogout
  features/filtros-salvos/    persistência de filtros por usuário/tela
```

Features presentes (16): auditoria-cnpj, auth, avisos, empresas, fila-jobhosts, filtros-salvos, home, logs-auditoria, onboards, permissoes (= Usuários do BackOffice), relatorio-engajamento, relatorio-uso-funcionalidades, shell, tokens-ia, trials, usuarios-clientes.

## Rotas (`router.tsx`)

Árvore: `rootRoute` → `loginRoute` (`/login`; quem já tem sessão é redirecionado a `/`) e `appRoute` (id `app`, layout `AppLayout`; `beforeLoad` consulta `/api/auth/me` e manda para `/login` sem sessão). Filhas de `appRoute`: `/` (Home, sem guard de tela), `/avisos`, `/avisos/leituras`, `/avisos/novo`, `/avisos/$id`, `/avisos/$id/acompanhamento`, `/ferramentas/{empresas,usuarios-clientes,trials,relatorio-engajamento,relatorio-uso-funcionalidades,auditoria-cnpj,logs-auditoria,onboards,fila-jobhosts}`, `/usuarios` (só admin), `/ia/tokens`, `/ia/tokens/usuarios`, `/ia/tokens/empresas`. Para adicionar rota: criar `createRoute` e incluir em `appRoute.addChildren([...])` (o pacote `@tanstack/router-plugin` está nas devDependencies, mas **não é usado** no `vite.config.ts`).

Guard de acesso (`features/auth/acesso.ts`): `exigirAcesso({telas:[...]} | {soAdmin:true})` lê o cache de `/api/auth/me` (staleTime 5 min); se negar, revalida uma vez (acesso recém-liberado) e então redireciona para `/` (sem permissão) ou `/login`. A API continua a defesa real (403). Os ids de tela (`telas`) vêm do backend (`TelasLiberadas`, ver `autenticacao-e-autorizacao.md`) e devem coincidir com `NAV_ITEMS[].tela`.

`acessoComFiltros(requisito, pathname)` = guard + restauração de filtros salvos. Telas de consulta usam; Trials, Tokens IA e Home não.

## Padrão de feature

- `use<Modulo>.ts`: `queryOptions`/`useMutation` do TanStack Query chamando `api.GET/POST('/api/...')` com tipos de `components['schemas'][...]`; trata 403 (mensagem padrão), invalida a query key do módulo após mutação, mostra `notifications` do Mantine.
- `busca<Modulo>.ts`: `validarBusca<Modulo>(search)` = `validateSearch` da rota; filtros, ordenação, página e item aberto vivem na **URL** (search params); valor inválido cai no padrão (nunca quebra).
- `<Modulo>Page.tsx` + `Filtros*.tsx`, `*Drawer.tsx`/`*Modal.tsx`, `rotulos.ts`, `formatar.ts`.
- Visibilidade de ações por tela extra: o componente lê `usuario.telas` (ex.: `trials-criar`, `trials-prorrogar`, `fila-jobhosts-controle`).
- Gráficos: `@mantine/charts` + recharts (engajamento, uso de funcionalidades, home). CSV gerado no cliente (`csv.ts` por módulo; `;`, UTF-8 BOM; uso-funcionalidades neutraliza injeção de fórmula).
- Editor de avisos: TipTap (`EditorMensagem.tsx`), seletor de emojis, prévia (`/api/avisos/previa`), DOMPurify.

## Filtros salvos (item 9890)

`features/filtros-salvos/`: guarda o search da rota no `localStorage` (chave `smark-filtros:<usuarioId>:<rota>`), por usuário e tela, e restaura quando a rota é aberta sem parâmetros (URL com parâmetros sempre vence). Mapa `TELAS_COM_FILTRO_SALVO` lista as telas e os parâmetros transitórios que NÃO são guardados (`page`, `empresa`, `usuario`...). Nova tela de consulta com filtro lembrado = acrescentar aqui e usar `acessoComFiltros`. `BotaoLimparFiltros` limpa. `localStorage` pode lançar (janela privada): código já protege.

## Como adicionar uma tela (checklist)

1. Backend pronto e `npm run codegen` rodado (ver `contrato-e-codegen.md`).
2. Criar `src/features/<modulo>/` com hook, `busca*.ts`, Page, componentes e testes.
3. Registrar a rota em `router.tsx` (com `exigirAcesso`/`acessoComFiltros` e `validateSearch`) e incluí-la em `appRoute.addChildren`.
4. Adicionar item em `NAV_ITEMS` (`features/shell/AppLayout.tsx`) com `tela:` igual ao id liberado no backend (ou `soAdmin`).
5. Garantir que o backend devolve o id da tela em `/api/auth/me` (`TelasLiberadas`) e que a policy existe.
6. Se tiver filtros persistentes, atualizar `TELAS_COM_FILTRO_SALVO`.
7. Rodar `npm run lint`, `npm run build` (`tsc -b` pega erro de tipo), `npm test`.

## Testes do frontend

Vitest sem jsdom (ambiente node): ~315 casos, majoritariamente lógica pura (`*.test.ts`) e renderização estática (`renderToStaticMarkup`) em alguns `*.test.tsx`. Não há testing-library nem testes e2e. **O CI não roda `npm test`** (só lint e build).

## Atenção

- 403 de tela tem texto padrão: "Você não tem permissão para acessar esta tela." (`MSG_SEM_PERMISSAO`).
- Sessão é estado de servidor: `/api/auth/me` (TanStack Query, 5 min); nada de token em JS.
- Datas: a API devolve instantes UTC (ISO 8601) em módulos novos; o front formata em `America/Sao_Paulo` (ex.: fila JobHosts, avisos); em módulos legados as datas vêm como gravadas (hora local sem fuso). Conferir módulo a módulo.
- `frontend/dist` é ignorado pelo git; só o `dotnet publish` o empacota.
