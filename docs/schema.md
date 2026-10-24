# Schema do banco

Mapa das tabelas, buckets e políticas do Supabase do Veronica Lens. Este
documento descreve o estado após as migrations em `supabase/migrations/`. Quando
uma migration mudar uma tabela, atualize a seção correspondente no mesmo PR.

Convenções:
- Chaves primárias são `uuid` com `gen_random_uuid()`, exceto `system_status`.
- Datas são `timestamptz` em UTC, com `created_at` e `updated_at` preenchidos
  pelo banco.
- Toda tabela exposta pela API tem RLS habilitado. Papéis: `anon`,
  `authenticated` (usuário logado) e `service_role` (servidor, ignora RLS).
- Admin é um usuário com `user_roles.role = 'admin'`. Admins leem e alteram
  dados de todos os usuários.

## Tipos enumerados

| Tipo | Valores |
| --- | --- |
| `app_role` | `user`, `admin` |
| `learning_progress_status` | `not_started`, `in_progress`, `completed` |
| `observation_media_type` | `image`, `video` |
| `device_platform` | `android`, `ios` |
| `notification_delivery_status` | `pending`, `sent`, `failed` |

## Tabelas por área

### Status do sistema

**`system_status`**: linha única (`id = 'supabase'`) com `status`
(`available` ou `maintenance`). Permite que o app mostre manutenção.
- Leitura: `anon` e `authenticated`, somente a linha `supabase`.
- Escrita: somente via migration ou painel.

### Perfil e acesso

**`profiles`**: um registro por usuário em `auth.users`. Guarda `display_name`
(1 a 100 caracteres) e `avatar_url`.
- Usuário lê, cria e atualiza o próprio perfil. Admin lê e atualiza todos.
- Exclusão somente por admin.

**`user_roles`**: papel de cada usuário (`user` ou `admin`). Padrão `user`.
- Usuário lê o próprio papel. Somente admin cria, altera e remove.

**`device_push_tokens`**: tokens de push por dispositivo (`android` ou `ios`),
com `active` e `last_seen_at`. `token` é único.
- Usuário gerencia os próprios tokens. Admin lê e altera todos.

**`event_preferences`**: preferências de aviso por tipo de evento, com
`enabled` e `lead_minutes` (0 a 43200). Única por usuário e tipo.
- Usuário gerencia as próprias preferências. Admin lê e altera todas.

### Aprendizado

**`learning_tracks`**: trilhas, com `slug` único, `title`, `description`,
`published` e `sort_order`.
- Usuário lê trilhas publicadas. Admin lê e gerencia todas.

**`lessons`**: lições de uma trilha, com `track_id`, `slug` e `content`. Únicas
por trilha em `slug` e em `sort_order`. Apagadas em cascata com a trilha.
- Mesma regra de leitura das trilhas: publicadas para usuários, todas para admin.

**`user_learning_progress`**: progresso de cada usuário em cada lição, com
`status` e `progress_percent` (0 a 100). Único por usuário e lição.
- `completed` exige `progress_percent = 100` e `completed_at` preenchido.
- Usuário lê, cria, atualiza e remove o próprio progresso. Admin tem acesso a todos.

### Eventos astronômicos

**`astronomical_events`**: eventos curados (eclipses, chuvas de meteoros,
conjunções), com `slug` único, `event_type`, `starts_at`, `ends_at` opcional e
`published`. Também guarda as colunas de sincronização usadas pelo job antigo.
- Usuário lê eventos publicados. Admin gerencia todos.

**`astronomical_event_sync_runs`**: registro de execuções do antigo job de
sincronização por localização, com `status` (`running`, `succeeded`, `failed`).
- Somente `service_role`.
- Marcada para limpeza: a remoção depende de migration própria.

**`notification_deliveries`**: registro de envio de push por evento e
dispositivo. Único por evento e token, o que evita envio duplicado.
- Somente `service_role` (leitura, inserção e atualização). `anon` e
  `authenticated` não têm acesso.

### Observações

**`observations`**: observações do usuário, com `title`, `notes`,
`celestial_object_name`, `observed_at`, latitude e longitude opcionais e
`astronomical_event_id` opcional (se o evento for apagado, o campo vira nulo).
- Usuário lê, cria, atualiza e apaga as próprias. Admin tem acesso a todas.

**`observation_media`**: arquivos de cada observação, com `storage_path` único,
`media_type` e `captured_at` opcional. A chave estrangeira composta
`(observation_id, user_id)` impede que a mídia aponte para observação de outro
usuário. Apagada em cascata com a observação.
- Mesma regra de acesso de `observations`.

## Storage

| Bucket | Público | Limite | Tipos aceitos | Caminho |
| --- | --- | --- | --- | --- |
| `avatars` | sim (leitura) | 5 MiB | JPEG, PNG, WebP | `<user_id>/avatar.<ext>` |
| `observations` | não | 50 MiB | JPEG, PNG, WebP, HEIC, HEIF, MP4, QuickTime | `<user_id>/<observation_id>/<photo_id>.<ext>` |

- `avatars`: qualquer um lê. Só o dono envia, altera ou apaga o próprio
  arquivo. Admin tem acesso a todos.
- `observations`: somente o dono e admin leem. A política exige que o primeiro
  segmento do caminho seja o `user_id` do usuário.

## Funções e privilégios

- Schema `private`: não exposto pela API. Guarda a função de atualização de
  `updated_at`, usada pelos triggers das tabelas.
- `notification_deliveries` tem acesso revogado de `anon` e `authenticated`
  (`revoke all`), com `select`, `insert` e `update` apenas para `service_role`.

## Relações principais

```text
auth.users
 ├── profiles (1:1)
 ├── user_roles (1:1)
 ├── device_push_tokens (1:N)
 ├── event_preferences (1:N)
 ├── user_learning_progress (1:N) ── lessons (N:1) ── learning_tracks
 └── observations (1:N)
        ├── observation_media (1:N)
        └── astronomical_events (N:1, opcional)

astronomical_events
 ├── notification_deliveries (N:N com device_push_tokens)
 └── astronomical_event_sync_runs (sem vínculo direto)
```

## Testes

Os testes pgTAP ficam em `supabase/tests/`, um por área:

| Arquivo | Área |
| --- | --- |
| `system_status_test.sql` | Status do sistema |
| `initial_domain_schema_test.sql` | Schema inicial |
| `private_data_rls_test.sql` | RLS de dados pessoais |
| `auth_role_claims_test.sql` | Papéis e claims |
| `profile_preferences_test.sql` | Perfil e preferências |
| `observation_capture_metadata_test.sql` | Metadados de captura |
| `astronomical_event_sync_test.sql` | Sincronização de eventos |
| `notification_deliveries_test.sql` | Ledger de push |
