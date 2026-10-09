---
name: run-crm-local
description: Sobe o SMark CRM em localhost (Redis + SiteExpress.SMark.BackOffice.Api + SiteExpress.SMark.UI) ou o BackOffice MVC. Use quando pedirem para rodar, executar, subir ou testar o CRM/aplicação web localmente.
---

# Rodar o SMark CRM localmente

O CRM web (`SiteExpress.SMark.UI`) **só funciona** com o Redis e a `SiteExpress.SMark.BackOffice.Api` rodando em paralelo. Todos são .NET Framework 4.8, hospedados no IIS Express.

Os caminhos abaixo são relativos à raiz do clone do repositório `smark-crm`. Se a sessão estiver em outro repositório, localize o clone primeiro, pela URL do remote, que termina em `/_git/smark-crm` (ex.: `W:\Smark\Projetos\smark-git`). Se não achar, pergunte o caminho ao usuário.

## 0. Ambiente do AWS Parameter Store (por desenvolvedor)

As conexões de banco e as demais configurações vêm do AWS Parameter Store. Cada desenvolvedor tem o seu ambiente lá, e a aplicação escolhe qual usar pela chave `AWSParameterStoreEnvironment` no config:

```xml
<add key="AWSParameterStoreEnvironment" value="env-localPedro" />
```

Os valores conhecidos são `env-localPedro`, `env-localCassio` e `env-localRegis`. Repare no prefixo `env-`.

1. Descubra de qual dev é a sessão (git user, e-mail ou o que o usuário disser). Se não der para saber, **pergunte**.
2. Confira a chave nos configs das aplicações que vão subir e ajuste se estiver com o ambiente de outra pessoa:
   - `src/SMark/Web/Web.config` (UI do CRM)
   - `src/Services/SiteExpress.SMark.BackOffice.Api/Web.config`
   - `src/BackOffice/SiteExpress.Smark.BackOffice.MVC.UI/Web.config` (se for subir o BackOffice MVC)
   - `src/SiteExpress.SMark.JobHost/App.config` (se for subir o JobHost)
3. Esses arquivos são **versionados**, e o valor muda quando algum dev commita o dele. Avise o usuário quando alterar e **não inclua essa mudança em commits**, a menos que ele peça.

## 1. Redis

Use sempre este executável (nunca outro Redis):

```
W:\Smark\redis-latest\redis-server.exe
```

- **Se o `.exe` não existir nesse caminho**, não procure outro Redis nem instale um. Pare e peça ao usuário o caminho correto do `redis-server.exe`.
- Antes de subir, veja se ele já está rodando: `Get-NetTCPConnection -LocalPort 6379 -ErrorAction SilentlyContinue`.
- Se não estiver, rode-o em background, com o diretório de trabalho em `W:\Smark\redis-latest`.

## 2. Build

Antes de subir, confira se os `bin` estão completos. Não basta existir a DLL do projeto. Veja também se há dependências como `BouncyCastle.Crypto.dll` e se o `bin` tem ~180+ DLLs:
- `src/SMark/Web/bin/SiteExpress.SMark.UI.dll`
- `src/Services/SiteExpress.SMark.BackOffice.Api/bin/SiteExpress.SMark.BackOffice.Api.dll`

Se o `bin` estiver incompleto ou desatualizado:
- Builde você mesmo pela linha de comando, mesmo com o Visual Studio aberto. Só não rode ao mesmo tempo que um build do VS. Se os `bin` ficarem vazios depois do build, investigue e informe o usuário.
- Use o MSBuild do VS (localize com `vswhere -latest -find "MSBuild\**\Bin\MSBuild.exe"`). **Não use `/restore`**: com `packages.config` ele falha com "O caminho tem um formato inválido". Os pacotes já estão em `src/packages`.

```
MSBuild.exe src/Services/SiteExpress.SMark.BackOffice.Api/SiteExpress.SMark.BackOffice.Api.csproj /p:Configuration=Debug /m /nodeReuse:false
MSBuild.exe src/SMark/Web/SiteExpress.SMark.UI.csproj /p:Configuration=Debug /m /nodeReuse:false
```

## 3. BackOffice.Api (porta 1999) e UI do CRM (porta 62662)

Os sites já estão definidos no `applicationhost.config` da solution. Suba cada um em background:

```
"C:\Program Files\IIS Express\iisexpress.exe" /config:src\.vs\Smark_SolutionCompleta\config\applicationhost.config /site:SiteExpress.SMark.BackOffice.Api
"C:\Program Files\IIS Express\iisexpress.exe" /config:src\.vs\Smark_SolutionCompleta\config\applicationhost.config /site:SiteExpress.SMark.UI
```

- A API: http://localhost:1999/. Um **404 na raiz é normal**, porque a Web API não tem rota em `/`.
- O CRM: http://localhost:62662/. O sucesso é um 200 com a tela de login ("Smark CRM - Login").
- A primeira requisição pode demorar (cold start do ASP.NET), então use um timeout alto.
- Se der 500 com corpo vazio, veja o Event Log (`Get-WinEvent` em Application, provider `ASP.NET 4.0.30319.0`). Geralmente é DLL faltando no `bin`.

Ordem: Redis → BackOffice.Api → UI.

## Outros projetos (independentes)

- **SiteExpress.Smark.BackOffice.MVC.UI** (BackOffice interno): `/site:SiteExpress.Smark.BackOffice.MVC.UI`, http://localhost:49981/. Não depende do CRM.
- **SiteExpress.SMark.JobHost** (serviço Windows, `src/SiteExpress.SMark.JobHost`): roda em paralelo, mas é independente. Só suba se a tarefa envolver jobs ou processamento em background.
