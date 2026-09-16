---
name: flutter-supabase-riverpod
description: Scaffolds and extends Flutter apps using Supabase for the backend, Riverpod for dependency injection and state, GoRouter for auth-aware navigation, and a feature-first repository/notifier architecture. Use when creating a new Flutter application, adding a CRUD feature, running supabase start, migrations, or Edge Functions, or when the user mentions Flutter + Supabase + Riverpod.
---

# Flutter + Supabase + Riverpod

Follow the templates in this skill. Every app uses this structure. Do not copy from another project for patterns.

## Architecture

```
Widget (UI)
    ↕  ref.watch / ref.read
Notifier (state + actions)
    ↕  ref.read / ref.watch
Repository (data) / Service (rules + shared I/O)
    ↕  constructor injection
SupabaseClient (via supabaseProvider)
```

| Layer | Owns | Must not |
|---|---|---|
| **Widget** | Layout, local form state, debounce, button loading | Call Supabase or repositories |
| **Notifier** | Server-backed state, search query, invalidation | Know table names or JSON keys |
| **Model** | Entity fields, `fromJson` / `toJson`, getters, setters, withers | Implement rules or calculations |
| **Repository** | Queries, JSON ↔ model, auth preconditions | Hold UI state or import widgets |
| **Service** | Entity business rules; I/O reused by many features | Hold UI state or import widgets |

Widgets never import `supabase_flutter`. Repositories never hold UI state. **No business logic on the model.**

## New app workflow

Copy this checklist and complete it in order:

```
New app:
- [ ] 1. Confirm name, org, package, first feature, home route
- [ ] 2. flutter create + dependencies + .env + .gitignore
- [ ] 3. supabase init + start + fill .env from status
- [ ] 4. Core: supabaseProvider, services, splash, theme
- [ ] 5. Auth: repository, StreamNotifier, login/signup
- [ ] 6. Router: GoRouter + auth redirect + splash
- [ ] 7. First feature: model → repository → notifier → screens
- [ ] 8. Register GoRoutes
- [ ] 9. README (local start, migrations, functions, env)
```

**Step 1 — Confirm scope** if the user did not specify:

- App display name and Dart package name
- Android/iOS org (reverse-DNS, e.g. `com.example` — do not assume)
- Whether email/password auth is required (default: yes)
- First feature name (singular domain noun, e.g. `note`, `task`)
- Whether that feature stores images
- Home route after login (default: that feature's index)

**Step 2 — Scaffold**

```bash
flutter create --org <org> --project-name <package> <app_dir>
```

Install dependencies with `flutter pub add` so pub resolves current versions. Do not pin versions in `pubspec.yaml` by hand.

```bash
flutter pub add flutter_riverpod flutter_dotenv supabase_flutter go_router
flutter pub add --dev custom_lint riverpod_lint
```

If the first feature stores images: `flutter pub add image_picker`.

`.env` (and `flutter: assets: - .env`):

```
SUPABASE_URL=
SUPABASE_PUBLISHABLE_KEY=
```

Also write `.env.example` with the same keys (empty values) and add `.env`, `supabase/.temp/`, and `supabase/.env` to `.gitignore`. Never commit `.env`.

Enable `custom_lint` in `analysis_options.yaml`. Templates: [bootstrap.md](bootstrap.md) (main, auth, router) and [ui.md](ui.md) (shared widgets, theme).

**Step 3 — Local Supabase** — follow [supabase.md](supabase.md): `supabase init`, `[auth.email] enable_confirmations = false`, `supabase start`, fill `.env` from `supabase status -o env`. Do not invent keys. Optional seed user after the first migration exists.

**Steps 4–6** — copy the bootstrap templates. Do not invent a different auth or redirect scheme.

**Step 7** — follow [feature.md](feature.md) for the first entity: Dart files **and** `supabase migration new` + `db reset`.

**Step 8** — register `GoRoute`s. If this is the home feature, use its index as `initialLocation` and as the post-login redirect target.

**Step 9** — write `README.md` from the template in [supabase.md](supabase.md).

Remote: link the **dev** project (`supabase link --project-ref`), then `supabase db push` and `supabase functions deploy` when the user is ready. Never link production by default.

## New feature workflow

1. Model (`fromJson` / `toJson`, snake_case keys)
2. Repository + `*RepositoryProvider` injecting `supabaseProvider` (and services)
3. Notifier: list = `AsyncNotifierProvider`; detail = `FutureProvider.family`
4. Screens: index / new / edit / view + form + list
5. Register `GoRoute`s
6. Never import `supabase_flutter` in widgets
7. `supabase migration new …`, put table/RLS/storage SQL in that file, `supabase db reset`
8. If the feature needs secrets or service-role logic: `supabase functions new <name>`, invoke from the repository, `supabase functions serve` locally

Full templates: [feature.md](feature.md). CLI: [supabase.md](supabase.md).

## Folder layout

```
lib/
  main.dart
  core/
    providers/          # supabaseProvider
    services/           # ImageStorageService, form validators
    screens/            # splash
    widgets/            # BodyContainer, inputs, dialogs, StoredImage
    enums/
  auth/
    repositories/       # AuthRepository + provider
    providers/          # AuthNotifier (StreamNotifier)
    screens/            # login, signup
    widgets/            # forms
  router/app_router.dart
  themes/
  <feature>/
    models/             # entity + fromJson/toJson + getters/setters/withers
    repositories/       # class + *RepositoryProvider
    providers/          # notifiers + family providers
    screens/            # index, new, edit, view
    widgets/            # list, list tile, form
    services/           # optional: entity business logic
    enums/              # optional
    controllers/        # optional: multi-step UI orchestration
    mappers/            # optional: external API → domain
supabase/
  config.toml
  migrations/           # source of truth for schema + RLS
  seeds/                # local only (`db reset`)
  functions/            # Edge Functions (`_shared/` + one folder per function)
```

One feature folder per domain. Shared I/O lives in `core/services/`. Entity business logic lives in `<feature>/services/`.

## Provider catalog

| Need | Type | Example |
|---|---|---|
| Immutable dependency | `Provider` | `supabaseProvider`, `*RepositoryProvider` |
| Session / continuous stream | `StreamNotifierProvider` | `authProvider` |
| List + search + mutations | `AsyncNotifierProvider` | `notesProvider` |
| One entity by id | `FutureProvider.family` | `noteProvider(id)` |

Do not use `StateProvider` / `ChangeNotifier` for server data. Do not put list state in the widget.

### `ref.watch` vs `ref.read`

| Use | Where |
|---|---|
| `ref.watch` | `build()` of a widget or provider — UI/provider should rebuild |
| `ref.read` | Clicks, submits, GoRouter `redirect`, one-off fetches |

`ref.watch` is illegal inside arbitrary callbacks. GoRouter `redirect` uses `ref.read(authProvider)`. Auth changes reach the router via `_AuthRefreshNotifier` (`ref.listen` → `notifyListeners` → `refreshListenable`).

## Hard rules

1. **Provider tree** — everything backend-related depends on `supabaseProvider`, never `Supabase.instance` in feature code (only the root provider and `main.dart` initialize it).
2. **Provider next to class** — `noteRepositoryProvider` lives in the repository file; the notifier provider lives in the notifier file.
3. **No business logic on models.** The model is the entity: fields, `fromJson` / `toJson`, getters, setters, withers (`copyWith`). No `json_serializable` / Freezed unless the user asks. Rules, eligibility, and calculations belong in a service.
4. **Strip write fields** — repositories remove `id`, `created_at`, `updated_at` before insert/update. Include `user_id` only on create.
5. **Create requires a session** — throw `AuthException` if `currentUser` is null.
6. **Invalidate in the notifier** — after create: `ref.invalidateSelf()`. After update/delete: also `ref.invalidate(entityProvider(id))` and any dependent family/list providers.
7. **Local UI stays local** — `TextEditingController`, form validation, debounce `Timer`, submit `_isLoading` use `setState`.
8. **Search** — widget debounces (~300 ms) and calls `notifier.setSearchQuery`; notifier stores the query and `invalidateSelf()`. Lists use `skipLoadingOnReload: true`.
9. **Routes live on screens** — `static const routeName`. Patterns: `/auth/login`, `/<feature>/index`, `/<feature>/new`, `/<feature>/:id/view`, `/<feature>/:id/edit`, `/splash`.
10. **Redirect order** — (1) `authState.isLoading` → splash (2) leaving splash → home or login (3) logged-out + not `/auth*` → login (4) logged-in + `/auth*` → home. Do not reorder.
11. **Images** — pick with `ImageInput`; upload via `ImageStorageService`; persist `image_path` (storage path); display with signed URLs (`StoredImage`). Path: `$userId/<timestamp>.<ext>`. On replace/delete, remove the previous storage object.
12. **External APIs** — public, keyless APIs: Dart repository + mapper; widgets see domain models only. Secrets, service-role writes, or third-party credentials: Edge Function; the feature repository invokes it.
13. **Imports** — `main.dart` uses `package:<app>/...`. Every other `lib/` file uses `/...` from `lib` (e.g. `/note/models/note_model.dart`). No `../`.
14. **Errors in UI** — generic SnackBar ("An error occurred. Please try again."). List `when(error:)` uses a generic message, not `$error`. After `await`, check `mounted`.
15. **Navigation** — `context.push` for new/edit/view, `context.pop` after successful save, `context.go` after delete.
16. **Schema** — every table, policy, and bucket is a file under `supabase/migrations/`. Apply with `supabase db reset` (local) or `supabase db push` (remote). Do not create schema only in the Dashboard.
17. **Edge Functions** — secrets, service-role writes, and third-party APIs live in `supabase/functions/<name>/index.ts`. Dart repositories call `_client.functions.invoke`. Widgets never invoke functions. Local: `supabase functions serve` (or restart the stack after adding a function). Remote: `supabase functions deploy` + `supabase secrets set`.

## Screen map

| Screen | Widget type | Body |
|---|---|---|
| Index | `StatelessWidget` | `Scaffold` + `AppBar` + `BodyContainer` + list + FAB → new |
| New | `StatelessWidget` | `BodyContainer` + scroll + form (no entity) |
| Edit | `ConsumerWidget` | `ref.watch(entityProvider(id)).when(...)` + same form with entity |
| View | `ConsumerWidget` | `ref.watch(entityProvider(id)).when(...)` + FAB → edit |
| List | `ConsumerStatefulWidget` | Search + `ref.watch(listProvider).when(...)` |
| Form | `ConsumerStatefulWidget` | Local controllers; `ref.read(listProvider.notifier)` for save/delete |

Index screens do **not** need `ConsumerWidget` — the list widget watches the provider.

Delete: confirm with a dialog. If the entity is referenced elsewhere, block delete and show those names. After delete, `context.go` to the index.

## Additional resources

- Bootstrap (main, auth, router, env): [bootstrap.md](bootstrap.md)
- Shared UI (inputs, dialogs, theme): [ui.md](ui.md)
- Feature templates (model, repo, notifier, screens, SQL): [feature.md](feature.md)
- Supabase CLI (init, start, migrations, functions, README): [supabase.md](supabase.md)
- Compact code snippets: [reference.md](reference.md)
