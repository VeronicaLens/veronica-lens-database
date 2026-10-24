# veronica-lens-database

Schema, migrations, seed e testes do banco Supabase do Veronica Lens.

Este repositorio e a unica fonte das alteracoes de banco. A API
(`veronica-lens-api`) e o app (`veronica-lens-mobile`) consomem o schema,
mas nao versionam migrations.

## Estrutura

```text
.github/workflows/
  database.yml            # CI: migrations, seed, db:lint e pgTAP em PR e push
  deploy-production.yml   # Deploy: aplica migrations pendentes no push para main
docs/
  schema.md               # Mapa de tabelas, RLS, buckets e testes
supabase/
  config.toml      # Configuracao do ambiente local (portas, Auth, Storage)
  migrations/      # Migrations SQL com timestamp, aplicadas em ordem
  tests/           # Testes pgTAP (schema, RLS, Storage, privilegios)
  seed.sql         # Dados ficticios apenas para desenvolvimento e testes
```

Para entender as tabelas e as políticas de acesso, leia
[docs/schema.md](docs/schema.md).

## Requisitos

- Node.js 20 ou superior e npm.
- Docker Desktop em execucao, para o ambiente local.

A Supabase CLI e instalada como dependencia de desenvolvimento, com versao
fixa no `package-lock.json`. Nao e preciso instala-la globalmente.

## Ambiente local

```powershell
npm ci
npm run db:start
```

`db:start` sobe o Supabase local e aplica todas as migrations e o `seed.sql`.
Para ver URLs e chaves locais:

```powershell
npm run db:status
```

Para usar o banco local na API, configure no `.env` da API a URL e a chave
publica exibidas por `db:status`. Sao chaves locais de desenvolvimento; nunca
use chaves de producao em arquivos versionados.

| Comando             | Funcao                                                    |
| ------------------- | --------------------------------------------------------- |
| `npm run db:start`  | Sobe o Supabase local                                     |
| `npm run db:status` | Mostra URLs e chaves locais                               |
| `npm run db:reset`  | Recria o banco local com migrations e seed                |
| `npm run db:lint`   | Verifica erros nas funcoes e no schema local              |
| `npm run db:test`   | Executa os testes pgTAP em `supabase/tests`               |
| `npm run db:new`    | Cria uma migration: `npm run db:new -- nome_da_mudanca`   |
| `npm run db:diff`   | Mostra diferencas do banco local em relacao as migrations |
| `npm run db:stop`   | Para o ambiente local                                     |

## Criar uma migration

1. `npm run db:new -- descricao_curta`.
2. Escreva o SQL no arquivo criado em `supabase/migrations/`.
3. Adicione ou atualize testes em `supabase/tests/`.
4. `npm run db:reset` e `npm run db:test`.

Nunca altere uma migration que ja foi aplicada em um ambiente compartilhado;
crie uma nova migration com a correcao. Tabelas expostas pela API devem ter
RLS habilitado e testes cobrindo o acesso de cada papel.

## Seed

`supabase/seed.sql` contem apenas dados ficticios de desenvolvimento.
Nunca copie dados de producao, dados pessoais, credenciais ou segredos para ele.

## Auth local

`config.toml` autoriza somente `veronicalens://auth/reset-password` como
redirecionamento adicional, o mesmo destino configurado na API
(`PASSWORD_RESET_REDIRECT_URL`). No projeto hospedado, a lista de Redirect URLs
e configurada no painel do Supabase.

## CI

O workflow `.github/workflows/database.yml` roda em pushes e pull requests para
`dev`, `qual` e `main`. Ele sobe um Supabase limpo, aplicando migrations e seed,
e executa `db:lint` e `db:test`.

O deploy nos projetos hospedados é feito por `deploy-production.yml`, descrito em "Ambientes e deploy".
Os nomes dos arquivos de migration são os mesmos do histórico já registrado nesses
projetos, por isso não renomeie migrations existentes.

## Ambientes e deploy

| Ambiente | Onde roda | Como as migrations chegam |
| --- | --- | --- |
| Local | Docker, via `npm run db:start` | `db:reset` aplica tudo do zero, com seed |
| Produção | Projeto Supabase hospedado | `deploy-production.yml`, no push para `main` |

`dev` e `qual` não fazem deploy. O projeto de produção recebe somente as
migrations pendentes, sem seed e sem testes pgTAP.

### Fluxo

1. PR para `main`: a CI sobe o ambiente local do zero e roda lint e pgTAP.
2. Após o merge em `main`, o job `deploy` entra no environment `production`.
   Se o environment exigir revisão, o deploy espera a aprovação.
3. O job faz `supabase link`, mostra `migration list --linked`, roda
   `db push --dry-run` e só então aplica com `db push --linked --yes`.
4. Se falhar, o log mostra a migration que parou. Nada é desfeito
   automaticamente.

Também é possível disparar manualmente em **Actions > Deploy production database
> Run workflow**.

### Segredos do environment `production`

Configure em **Settings > Environments > production**. Não coloque esses valores
no repositório.

| Segredo | O que é |
| --- | --- |
| `SUPABASE_ACCESS_TOKEN` | Token pessoal da conta Supabase com acesso ao projeto |
| `SUPABASE_PROJECT_REF` | Identificador do projeto hospedado (aparece na URL do painel) |
| `SUPABASE_DB_PASSWORD` | Senha do Postgres do projeto hospedado |

### Antes do primeiro deploy automático

Confira se o histórico do projeto hospedado bate com o repositório. Depois de
`npx supabase link --project-ref <ref>`, rode:

```powershell
npx supabase migration list --linked
```

Se houver migration aplicada no remoto e ausente no repositório, ou o contrário,
resolva antes de depender do deploy. Use `supabase migration repair` somente
depois de entender a diferença.

## Alterar o banco em produção

- Produção muda somente por migration. Não altere tabelas, políticas ou buckets
  pelo painel do Supabase, porque isso cria divergência e pode quebrar o próximo
  deploy.
- Migrations não são revertidas automaticamente. Para desfazer, crie uma nova
  migration com a correção.
- Não edite uma migration já aplicada em produção.
- Mudanças destrutivas (apagar coluna ou tabela) devem ir em PR separado, com a
  justificativa descrita.