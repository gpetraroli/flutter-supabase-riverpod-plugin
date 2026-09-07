# Quick reference

## Provider tree

```
supabaseProvider
├── authRepositoryProvider → authProvider (StreamNotifier)
├── imageStorageServiceProvider
│   └── <feature>RepositoryProvider → <features>Provider (AsyncNotifier)
│                                  → <feature>Provider (FutureProvider.family)
└── routerProvider  (reads authProvider; does not watch supabase)
```

## Inject

```dart
final xRepositoryProvider = Provider<XRepository>((ref) {
  return XRepository(ref.watch(supabaseProvider));
});
```

## List + detail

```dart
final itemsProvider =
    AsyncNotifierProvider<ItemsNotifier, List<ItemModel>>(ItemsNotifier.new);

final itemProvider = FutureProvider.family<ItemModel, String>((ref, id) {
  return ref.watch(itemRepositoryProvider).fetchById(id);
});
```

`build()` / provider bodies: `ref.watch`. Mutation methods: `ref.read`.

## UI

```dart
final state = ref.watch(itemsProvider);
state.when(
  skipLoadingOnReload: true,
  data: (items) => ListView.builder(/* … */),
  error: (error, _) => const Text('An error occurred. Please try again.'),
  loading: () => const Center(child: CircularProgressIndicator()),
);

await ref.read(itemsProvider.notifier).createItem(/* … */);
```

## Auth action (no navigation)

```dart
await ref.read(authProvider.notifier).signIn(email, password);
// GoRouter redirect sends the user home
```

```dart
final session = await ref.read(authProvider.notifier).signUp(email, password);
if (session == null) {
  // Confirm email is enabled — tell the user to check their inbox
}
```

```dart
ref.read(authProvider.notifier).signOut();
```

## Search (widget)

```dart
_debounce?.cancel();
_debounce = Timer(const Duration(milliseconds: 300), () {
  if (!mounted) return;
  ref.read(itemsProvider.notifier).setSearchQuery(value);
});
```

## Invalidation

```dart
ref.invalidateSelf();                 // inside the list notifier
ref.invalidate(itemProvider(id));     // detail cache
ref.invalidate(otherFeatureProvider); // dependents
```

Keep invalidation inside the notifier that owns the mutation.

## Router snapshot

```dart
final authState = ref.read(authProvider);
final isLoggedIn = authState.value != null;
```

`_AuthRefreshNotifier` listens to `authProvider` and is passed as `refreshListenable`.

Redirect order: loading → splash; leave splash → home/login; guest on app route → login; user on `/auth*` → home.

## Images

- Form: `ImageInput` holds `ImageInputValue?` (bytes) until save
- Repository: `ImageStorageService.upload` returns a **path**; column `image_path`
- Display: `StoredImage(bucket: …, path: model.imagePath)` (signed URL)
- Replace/delete: remove the previous storage object

## Form submit skeleton

```dart
Future<void> _submit() async {
  if (!_formKey.currentState!.validate()) return;
  setState(() => _isLoading = true);
  try {
    if (widget.isEditForm) {
      await ref.read(itemsProvider.notifier).updateItem(/* … */);
    } else {
      await ref.read(itemsProvider.notifier).createItem(/* … */);
    }
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Saved successfully.')),
    );
    context.pop();
  } catch (_) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('An error occurred. Please try again.')),
    );
  } finally {
    if (mounted) setState(() => _isLoading = false);
  }
}
```

## Embed select (detail fetch)

```dart
.select('*, children!parent_id(*, related(*))')
```

Parse nested lists in the parent model's `fromJson`. Do not fetch children from the widget.

## Imports

```dart
// main.dart only
import 'package:<app>/router/app_router.dart';

// every other lib file
import '/core/providers/supabase_provider.dart';
import '/note/models/note_model.dart';
```

## Local vs provider state

| Widget `setState` | Notifier |
|---|---|
| Controllers, validation | Server lists |
| Debounce timer | Search query that hits the API |
| Submit button loading | Create / update / delete |
| Image picker bytes before save | Cache invalidation |

## Schema

```sh
supabase migration new add_<change>
supabase db reset          # local
supabase db push           # linked remote
```

Do not create tables only in the Dashboard. Keys: `supabase status -o env` → `SUPABASE_URL` / `SUPABASE_PUBLISHABLE_KEY`.

## Edge Function invoke (repository)

```dart
final response = await _client.functions.invoke(
  'do_something',
  body: {'id': id},
);
```

Local: `supabase functions serve`. Remote: `supabase functions deploy`.

## Anti-patterns

- `Supabase.instance.client` inside a widget or notifier
- Watching a provider in `onPressed` / `redirect`
- `ref.read` inside a provider `build()`
- Putting `AsyncValue` loading for submit on the list provider (use local `_isLoading`)
- Public storage URLs persisted in the database
- Committing `.env` or `supabase/.env`
- Creating tables only in the Dashboard (no migration file)
- `supabase link` to production during daily work
- Table or storage without RLS policies
- Invoking Edge Functions from a widget
- Putting third-party secrets in the Flutter app
- Code-generated models by default
- Feature widgets importing another feature's repository (go through that feature's notifier, or a dedicated repository method called by the owning notifier)
