# Supabase CLI (local + migrations)

Schema lives in `supabase/migrations/`. The repo is the source of truth. Do not create tables only in the Dashboard.

Requirements: Docker, [Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started).

```sh
npm i -g supabase
supabase login
```

## New app — init

From the app root, after `flutter create`:

```sh
supabase init
```

This creates `supabase/config.toml`. Then:

1. Set `[auth.email] enable_confirmations = false` so local signup creates a session.
2. Keep `[db.seed] enabled = true` and `sql_paths = ["./seeds/*.sql"]`.
3. Add `supabase/.temp/` to `.gitignore`.
4. Start and fill `.env` from local keys (do not invent them):

```sh
supabase start
supabase status -o env
```

Map:

| `supabase status` | `.env` |
|---|---|
| `API_URL` | `SUPABASE_URL` |
| `ANON_KEY` | `SUPABASE_PUBLISHABLE_KEY` |

Local URL is `http://127.0.0.1:54321`. On a physical Android device:

```sh
adb reverse tcp:54321 tcp:54321
```

First feature: write migrations per [feature.md](feature.md), then `supabase db reset`. Optional seed (below) after that.

Write `README.md` using the template at the bottom of this file.

## New feature — migration

Never paste SQL for the user to run by hand. Version it:

```sh
supabase migration new add_<what_changed>
```

Edit `supabase/migrations/<timestamp>_add_<what_changed>.sql`. Apply locally:

```sh
supabase db reset
```

Reset drops the local DB, reapplies every migration, then runs `supabase/seeds/`. Fix the SQL until reset succeeds. Commit the migration with the Dart code.

When the user is ready for the **linked remote**:

```sh
supabase db push
```

## Migrations (local ↔ remote)

| Target | Command | What it does |
|---|---|---|
| **Local** | `supabase start` | Start Docker stack; apply migrations on first start |
| **Local** | `supabase db reset` | Drop local DB, reapply all migrations + seeds |
| **Remote** | `supabase db push` | Apply pending migration files to the linked project |
| **Local → repo** | `supabase db diff -f <name>` | Generate a migration from Studio / ad-hoc local SQL |
| **Remote → repo** | `supabase db pull` | Baseline from an existing cloud DB (catch-up only) |

`supabase migration list` shows what is applied locally vs remote.

**Default (write SQL in the repo):** `migration new` → edit file → `db reset` → commit → `db push`. Do not change remote schema only in the Dashboard; history will drift.

**If you prototyped in local Studio:** `db diff -f describe_change` → review the file → `db reset` → commit → `db push`.

**Link.** Link the **dev** project, not production:

```sh
supabase link --project-ref <dev-project-id>
```

If you must push production, link prod, `db push`, then **link back to dev**.

## Seeds

`supabase/seeds/` runs only on `db reset` (local). Use for a confirmed email user and sample rows. Token columns on `auth.users` must be empty strings, not NULL.

`supabase/seeds/00_extensions.sql`:

```sql
create extension if not exists "pgcrypto";
```

`supabase/seeds/01_user.sql` (ask the user for email/password, or use `dev@example.com` / `password123`):

```sql
do $$
declare
  v_user_id uuid := 'a0000000-0000-4000-8000-000000000001';
  v_email text := 'dev@example.com';
  v_encrypted_pw text := crypt('password123', gen_salt('bf'));
begin
  insert into auth.users (
    id, instance_id, aud, role, email, encrypted_password,
    email_confirmed_at, recovery_sent_at, last_sign_in_at,
    raw_app_meta_data, raw_user_meta_data, created_at, updated_at,
    confirmation_token, email_change, email_change_token_new,
    recovery_token, email_change_token_current, is_sso_user, is_anonymous
  ) values (
    v_user_id,
    '00000000-0000-0000-0000-000000000000',
    'authenticated', 'authenticated', v_email, v_encrypted_pw,
    now(), now(), now(),
    '{"provider":"email","providers":["email"]}',
    format('{"email":"%s","email_verified":true,"phone_verified":false,"sub":"%s"}', v_email, v_user_id)::jsonb,
    now(), now(),
    '', '', '', '', '', false, false
  );

  insert into auth.identities (
    id, user_id, provider_id, identity_data, provider,
    last_sign_in_at, created_at, updated_at
  ) values (
    v_user_id, v_user_id, v_user_id::text,
    format('{"sub":"%s","email":"%s","email_verified":true,"phone_verified":false}', v_user_id, v_email)::jsonb,
    'email', now(), now(), now()
  );
end $$;
```

## CLI cheat sheet

| Command | Description |
|---|---|
| `supabase start` / `stop` | Local stack (needs Docker) |
| `supabase status` / `status -o env` | Local URL and keys |
| `supabase migration new <name>` | Empty file under `supabase/migrations/` |
| `supabase migration list` | Local vs remote history |
| `supabase db reset` | Local: wipe + all migrations + seeds |
| `supabase db push` | Remote: pending migrations |
| `supabase db diff -f <name>` | Local DB changes → migration file |
| `supabase db pull` | Remote schema → migration (baseline) |
| `supabase db shell` | `psql` on local |
| `supabase link --project-ref <ref>` | Link repo to a remote project |
| `supabase functions new <name>` | Scaffold `supabase/functions/<name>/index.ts` |
| `supabase functions serve` | Serve functions against the local stack |
| `supabase functions deploy` / `deploy <name>` | Deploy to the linked remote |
| `supabase secrets set KEY=value` | Remote function secrets (not values in git) |

## Edge Functions

Use an Edge Function when the work cannot live in a Dart repository + RLS:

- Third-party APIs or secrets
- Service-role writes that must not use the anon key
- Multi-step server logic (notify many users, start a workflow)

Table CRUD with RLS stays in the Dart repository. Widgets never call `functions.invoke`.

### Layout

```
supabase/functions/
  README.md                 # list functions + required secrets
  _shared/                  # shared Deno modules
  <function_name>/index.ts  # one folder per function, snake_case
```

```sh
supabase functions new <function_name>
```

JWT verification stays on (`verify_jwt: true`). After adding a function, reload locally:

```sh
supabase stop && supabase start
# or, while iterating:
supabase functions serve
```

Local URL: `http://127.0.0.1:54321/functions/v1/<function_name>`. Extra secrets: `supabase/.env` (gitignored; `SUPABASE_URL` / anon / service role are injected). Remote:

```sh
supabase secrets set MY_SECRET=value
supabase functions deploy <function_name>
# or all:
supabase functions deploy
```

Keep `supabase/functions/README.md` updated (name, purpose, secrets). Stub:

```markdown
# Edge functions

| Name | Purpose |
|------|---------|
| `<name>` | … |

## Secrets

Set remotely with `supabase secrets set`. Local: `supabase/.env` (not committed).
```

### Function template — `supabase/functions/<name>/index.ts`

```ts
import { createClient } from "jsr:@supabase/supabase-js@2";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
};

function json(body: unknown, status = 200): Response {
  return new Response(JSON.stringify(body), {
    headers: { ...corsHeaders, "Content-Type": "application/json" },
    status,
  });
}

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const authHeader = req.headers.get("Authorization");
    if (!authHeader) return json({ error: "Missing authorization header" }, 401);

    const supabaseUrl = Deno.env.get("SUPABASE_URL");
    const supabaseAnonKey = Deno.env.get("SUPABASE_ANON_KEY");
    const serviceRoleKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY");
    if (!supabaseUrl || !supabaseAnonKey || !serviceRoleKey) {
      throw new Error("Supabase environment variables are not configured.");
    }

    const supabaseUser = createClient(supabaseUrl, supabaseAnonKey, {
      global: { headers: { Authorization: authHeader } },
    });
    const {
      data: { user },
      error: userError,
    } = await supabaseUser.auth.getUser();
    if (userError || !user) return json({ error: "Unauthorized" }, 401);

    const body = await req.json();
    // validate body…

    const supabaseAdmin = createClient(supabaseUrl, serviceRoleKey);
    // privileged work with supabaseAdmin; user-scoped work with supabaseUser

    return json({ ok: true });
  } catch (error) {
    return json({ error: `${error}` }, 500);
  }
});
```

Shared helpers go in `supabase/functions/_shared/` and are imported with relative paths (`../_shared/foo.ts`).

### Dart — invoke from the repository

```dart
final response = await _client.functions.invoke(
  'do_something',
  body: {'id': id},
);
final data = response.data;
if (data is Map && data['error'] != null) {
  throw Exception(data['error'].toString());
}
```

Catch `FunctionException`. Map errors to a generic UI message in the widget. Do not use `Supabase.instance` — inject `supabaseProvider`.

## README.md template

Write this at the app root when scaffolding. Replace `<AppTitle>`.

~~~~markdown
# <AppTitle>

Flutter + Supabase + Riverpod.

## Requirements

- Flutter SDK matching `pubspec.yaml` (`environment.sdk`).
- Docker (local Supabase).
- [Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started).

## Development

### 1. Supabase CLI

```sh
npm i -g supabase
supabase login
supabase link --project-ref <dev-project-id>
```

Link the **dev** project, not production.

### 2. Start locally

```sh
supabase start
```

First start applies `supabase/migrations/`. Use `supabase db reset` for a clean DB (migrations + `supabase/seeds/`).

### 3. Environment

Create `.env` at the project root (`supabase status -o env`):

| Variable | Description |
|----------|-------------|
| `SUPABASE_URL` | API URL (`http://127.0.0.1:54321` locally) |
| `SUPABASE_PUBLISHABLE_KEY` | Anon / publishable key |

Never commit `.env`. Copy `.env.example` for the keys. Do not commit `supabase/.env` (local function secrets).

Typical files: `.env` (local), `.env_remote` (linked dev), `.env_prod` (production).

### 4. Run the app

```sh
flutter pub get
flutter run
```

Physical Android + local API:

```sh
adb reverse tcp:54321 tcp:54321
flutter run
```

## Migrations

SQL files in `supabase/migrations/` are the source of truth.

| Target | Command |
|--------|---------|
| Local | `supabase db reset` |
| Remote | `supabase db push` |
| Local DB → new file | `supabase db diff -f describe_change` |
| Remote → new file | `supabase db pull` (baseline only) |

Default: `supabase migration new <name>` → edit SQL → `db reset` → commit → `db push`.

Do not edit remote schema only in the Dashboard.

```
Repo (supabase/migrations/*.sql)
        ├─ db reset  → Local DB
        ├─ db push   → Remote DB
        └─ db diff   → New migration file
```

## Deploy to remote

1. `supabase db push` (linked **dev**).
2. `supabase functions deploy` (and `supabase secrets set` for any extra secrets).
3. Point `.env_remote` at the remote URL and publishable key.
4. Production: `supabase link --project-ref <prod-project-id>`, `db push`, `functions deploy`, then **link back to dev**.

## Edge Functions

Source of truth: `supabase/functions/`. See `supabase/functions/README.md` for names and secrets.

```sh
supabase functions new <name>
supabase functions serve
supabase functions deploy <name>
```

After adding a function folder, `supabase stop && supabase start` (or `functions serve`) so `http://127.0.0.1:54321/functions/v1/<name>` exists.

The app calls functions from Dart repositories (`functions.invoke`), never from widgets.
~~~~
