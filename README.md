# veronica-lens-database

Schema, migrations, seed e testes do banco Supabase do Veronica Lens.

Este repositorio e a unica fonte das alteracoes de banco. A API
(`veronica-lens-api`) e o app (`veronica-lens-mobile`) consomem o schema,
mas nao versionam migrations.

## Estrutura

```text
supabase/
  config.toml      # Configuracao do ambiente local (portas, Auth, Storage)
  migrations/      # Migrations SQL com timestamp, aplicadas em ordem
  tests/           # Testes pgTAP (schema, RLS, Storage, privilegios)
  seed.sql         # Dados ficticios apenas para desenvolvimento e testes
```

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

As migrations ainda nao sao aplicadas automaticamente nos projetos hospedados.
Elas vieram de `veronica_lens_api` com os mesmos nomes de arquivo, para manter
o historico de migrations ja registrado nesses projetos.