# Flutter + Supabase + Riverpod

Agent skill for scaffolding Flutter apps with Supabase, Riverpod, and GoRouter.

This package is an [Agent Plugin](https://agent-plugins.org/) (Cursor, Copilot, VS Code, …) and a [Claude Code](https://code.claude.com/docs/en/plugins) plugin. The skill lives in `skills/flutter-supabase-riverpod/`.

Use it to:

- Create a new Flutter app with email/password auth, GoRouter, and a first CRUD feature
- Add another feature (model → repository → notifier → screens)
- Run local Supabase, migrations, and Edge Functions

After installing, invoke it in chat with `/flutter-supabase-riverpod`.

## Cursor

Install in **one** of these ways:

### Marketplace

Open **Customize**, search `flutter-supabase-riverpod`, then **Install** (user or project scope).

On a Teams or Enterprise plan, you can also import this GitHub repo as a team marketplace under **Dashboard → Plugins**.

### Manual

Symlink this repo into Cursor’s local plugins folder, then reload the window:

```sh
mkdir -p ~/.cursor/plugins/local
ln -s /absolute/path/to/flutter-supabase-riverpod-plugin ~/.cursor/plugins/local/flutter-supabase-riverpod
```

## Claude Code

Install in **one** of these ways:

### Marketplace

Add a marketplace that points at this repo, then:

```sh
claude plugin install flutter-supabase-riverpod@<marketplace-name>
```

### Manual

From this directory:

```sh
claude plugin validate .
claude --plugin-dir /absolute/path/to/flutter-supabase-riverpod-plugin
```
