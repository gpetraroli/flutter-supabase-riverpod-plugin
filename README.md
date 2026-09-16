# Flutter + Supabase + Riverpod

Agent skill for scaffolding Flutter apps with Supabase, Riverpod, and GoRouter.

This package is an [Agent Plugin](https://agent-plugins.org/) (Cursor, Copilot, VS Code, …) and a [Claude Code](https://code.claude.com/docs/en/plugins) plugin. The skill lives in `skills/flutter-supabase-riverpod/`.

Use it to scaffold or extend an app that follows a feature-first repository/notifier architecture:

```
Widget (UI)
    ↕  ref.watch / ref.read
Notifier (state + actions)
    ↕  ref.read / ref.watch
Repository (data) / Service (rules + shared I/O)
    ↕  constructor injection
SupabaseClient (via supabaseProvider)
```

Widgets own layout and local form state; they never import `supabase_flutter` or call repositories. Notifiers own server-backed state and do not know table names or JSON keys. Models are the entity (fields, `fromJson` / `toJson`, getters, setters, withers) and must not contain business logic. Repositories own queries and JSON ↔ model mapping. Services own entity rules and shared I/O (for example storage).

Riverpod is the dependency injection and state layer. `supabaseProvider` is the single backend root: feature code never uses `Supabase.instance`. Repositories and services take their dependencies in the constructor (`SupabaseClient`, other services) and are exposed as `Provider`s next to the class (`noteRepositoryProvider`). Notifiers and widgets receive those through `ref.watch` / `ref.read`. Immutable deps use `Provider`; session uses `StreamNotifierProvider`; lists use `AsyncNotifierProvider`; a single entity by id uses `FutureProvider.family`.

Typical tasks:

- Create a new Flutter app with email/password auth, GoRouter (auth redirect + splash), and a first CRUD feature
- Add another feature as model → repository → notifier → screens (entity rules go in a feature service)
- Run local Supabase, migrations, and Edge Functions

After installing, invoke it in chat with `/flutter-supabase-riverpod`.

## Cursor

Symlink this repo into Cursor’s local plugins folder, then reload the window:

```sh
mkdir -p ~/.cursor/plugins/local
ln -s /absolute/path/to/flutter-supabase-riverpod-plugin ~/.cursor/plugins/local/flutter-supabase-riverpod
```

## Claude Code

From this directory:

```sh
claude plugin validate .
claude --plugin-dir /absolute/path/to/flutter-supabase-riverpod-plugin
```
