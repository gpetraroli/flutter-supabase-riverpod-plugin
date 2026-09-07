# Flutter + Supabase + Riverpod

Agent skill for scaffolding Flutter apps with Supabase, Riverpod, and GoRouter.

This package is an [Agent Plugin](https://agent-plugins.org/) (Cursor, Copilot, VS Code, …) and a [Claude Code](https://code.claude.com/docs/en/plugins) plugin. The skill lives in `skills/flutter-supabase-riverpod/`.

## Cursor

Symlink into local plugins, then reload the window:

```sh
mkdir -p ~/.cursor/plugins/local
ln -s /absolute/path/to/flutter-supabase-riverpod-plugin ~/.cursor/plugins/local/flutter-supabase-riverpod
```

In chat: `/flutter-supabase-riverpod`

To publish: push this repo and submit the GitHub URL at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish), or import it into a team marketplace.

## Claude Code

From this directory:

```sh
claude plugin validate .
claude --plugin-dir /absolute/path/to/flutter-supabase-riverpod-plugin
```

Or add a marketplace that points at this repo, then:

```sh
claude plugin install flutter-supabase-riverpod@<marketplace-name>
```

Invoke with `/flutter-supabase-riverpod`.
