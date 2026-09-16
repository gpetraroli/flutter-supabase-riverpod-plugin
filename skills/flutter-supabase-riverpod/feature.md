# Adding a CRUD feature

Example entity: `Note` / table `notes`. Rename consistently (`note`, `notes`, `NoteModel`, `NoteRepository`, `notesProvider`, `noteProvider`).

Create files in this order. Register routes last.

## 1. Model — `lib/note/models/note_model.dart`

Manual `fromJson` / `toJson`. JSON keys are snake_case. Put derived values on the model (getters), not in widgets.

`imagePath` is a **storage path** (`userId/timestamp.jpg`), never a public URL. The column is `image_path`.

```dart
class NoteModel {
  const NoteModel({
    required this.id,
    required this.userId,
    required this.title,
    this.body,
    this.imagePath,
    required this.createdAt,
    required this.updatedAt,
  });

  final String id;
  final String userId;
  final String title;
  final String? body;
  final String? imagePath;
  final DateTime createdAt;
  final DateTime updatedAt;

  factory NoteModel.fromJson(Map<String, dynamic> json) {
    return NoteModel(
      id: json['id'] as String,
      userId: json['user_id'] as String,
      title: json['title'] as String,
      body: json['body'] as String?,
      imagePath: json['image_path'] as String?,
      createdAt: DateTime.parse(json['created_at'] as String),
      updatedAt: DateTime.parse(json['updated_at'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'user_id': userId,
      'title': title,
      'body': body,
      'image_path': imagePath,
      'created_at': createdAt.toIso8601String(),
      'updated_at': updatedAt.toIso8601String(),
    };
  }
}
```

Nested relations: parse optional join keys in `fromJson` (e.g. `json['items']`). Use a Supabase embed select in the repository, not a second client call from the widget.

Enums: store `.name` in JSON; read with `MyEnum.values.byName(json['col'])`.

## 2. Repository — `lib/note/repositories/note_repository.dart`

```dart
import 'dart:typed_data';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import '/core/providers/supabase_provider.dart';
import '/core/services/image_storage_service.dart';
import '/note/models/note_model.dart';

final noteRepositoryProvider = Provider<NoteRepository>((ref) {
  return NoteRepository(
    ref.watch(supabaseProvider),
    ref.watch(imageStorageServiceProvider),
  );
});

class NoteRepository {
  NoteRepository(this._client, this._imageStorage);

  final SupabaseClient _client;
  final ImageStorageService _imageStorage;

  static const _table = 'notes';
  static const imageBucket = 'note-images';

  Future<List<NoteModel>> fetchAll() async {
    final response = await _client
        .from(_table)
        .select()
        .order('created_at', ascending: false);

    return (response as List)
        .map((json) => NoteModel.fromJson(json as Map<String, dynamic>))
        .toList();
  }

  Future<List<NoteModel>> search(String query) async {
    final trimmed = query.trim();
    if (trimmed.isEmpty) return fetchAll();

    final pattern = '%$trimmed%';
    final response = await _client
        .from(_table)
        .select()
        .or('title.ilike.$pattern,body.ilike.$pattern')
        .order('created_at', ascending: false);

    return (response as List)
        .map((json) => NoteModel.fromJson(json as Map<String, dynamic>))
        .toList();
  }

  Future<NoteModel> fetchById(String id) async {
    final response =
        await _client.from(_table).select().eq('id', id).single();
    return NoteModel.fromJson(response);
  }

  Future<NoteModel> create({
    required String title,
    String? body,
    Uint8List? imageBytes,
    String imageExtension = 'jpg',
  }) async {
    final userId = _client.auth.currentUser?.id;
    if (userId == null) {
      throw const AuthException('User must be signed in to create a note.');
    }

    String? imagePath;
    if (imageBytes != null) {
      imagePath = await _imageStorage.upload(
        bucket: imageBucket,
        imageBytes: imageBytes,
        imageExtension: imageExtension,
      );
    }

    final response = await _client
        .from(_table)
        .insert(_writeJson(
          NoteModel(
            id: '',
            userId: userId,
            title: title,
            body: body,
            imagePath: imagePath,
            createdAt: DateTime.now(),
            updatedAt: DateTime.now(),
          ),
          includeUserId: true,
        ))
        .select()
        .single();

    return NoteModel.fromJson(response);
  }

  Future<NoteModel> update({
    required String id,
    required String title,
    String? body,
    Uint8List? imageBytes,
    String imageExtension = 'jpg',
    bool removeImage = false,
  }) async {
    final existing = await fetchById(id);
    var resolvedPath = existing.imagePath;

    if (imageBytes != null) {
      resolvedPath = await _imageStorage.upload(
        bucket: imageBucket,
        imageBytes: imageBytes,
        imageExtension: imageExtension,
      );
      await _deleteStoragePath(existing.imagePath);
    } else if (removeImage) {
      await _deleteStoragePath(existing.imagePath);
      resolvedPath = null;
    }

    final response = await _client
        .from(_table)
        .update(_writeJson(
          NoteModel(
            id: id,
            userId: existing.userId,
            title: title,
            body: body,
            imagePath: resolvedPath,
            createdAt: existing.createdAt,
            updatedAt: DateTime.now(),
          ),
        ))
        .eq('id', id)
        .select()
        .single();

    return NoteModel.fromJson(response);
  }

  Future<void> delete(String id) async {
    final existing = await fetchById(id);
    await _client.from(_table).delete().eq('id', id);
    await _deleteStoragePath(existing.imagePath);
  }

  Future<void> _deleteStoragePath(String? path) async {
    if (path == null || path.isEmpty) return;
    await _imageStorage.delete(bucket: imageBucket, path: path);
  }

  Map<String, dynamic> _writeJson(
    NoteModel note, {
    bool includeUserId = false,
  }) {
    final json = note.toJson()
      ..remove('id')
      ..remove('created_at')
      ..remove('updated_at');

    if (!includeUserId) {
      json.remove('user_id');
    }

    return json;
  }
}
```

Skip `ImageStorageService` if the feature has no images.

Related child rows (tags, line items): the repository owns the sync. After writing the parent, delete existing children and insert the new set in one place.

## 3. Notifier — `lib/note/providers/notes_provider.dart`

```dart
import 'dart:typed_data';

import 'package:flutter_riverpod/flutter_riverpod.dart';

import '/note/models/note_model.dart';
import '/note/repositories/note_repository.dart';

final notesProvider =
    AsyncNotifierProvider<NotesNotifier, List<NoteModel>>(NotesNotifier.new);

final noteProvider = FutureProvider.family<NoteModel, String>((ref, id) {
  return ref.watch(noteRepositoryProvider).fetchById(id);
});

class NotesNotifier extends AsyncNotifier<List<NoteModel>> {
  String _searchQuery = '';

  @override
  Future<List<NoteModel>> build() {
    return ref.watch(noteRepositoryProvider).search(_searchQuery);
  }

  void setSearchQuery(String query) {
    if (_searchQuery == query) return;
    _searchQuery = query;
    ref.invalidateSelf();
  }

  Future<void> createNote({
    required String title,
    String? body,
    Uint8List? imageBytes,
    String imageExtension = 'jpg',
  }) async {
    await ref.read(noteRepositoryProvider).create(
          title: title,
          body: body,
          imageBytes: imageBytes,
          imageExtension: imageExtension,
        );
    ref.invalidateSelf();
  }

  Future<void> updateNote({
    required String id,
    required String title,
    String? body,
    Uint8List? imageBytes,
    String imageExtension = 'jpg',
    bool removeImage = false,
  }) async {
    await ref.read(noteRepositoryProvider).update(
          id: id,
          title: title,
          body: body,
          imageBytes: imageBytes,
          imageExtension: imageExtension,
          removeImage: removeImage,
        );
    ref.invalidateSelf();
    ref.invalidate(noteProvider(id));
  }

  Future<void> deleteNote(String id) async {
    await ref.read(noteRepositoryProvider).delete(id);
    ref.invalidateSelf();
    ref.invalidate(noteProvider(id));
  }
}
```

Use `ref.watch` in `build()` / provider bodies; `ref.read` in mutation methods.

If another feature caches this data, invalidate that provider from this notifier after update/delete.

Use `StreamNotifierProvider` only for a continuous backend stream (auth). Do not stream a list unless the product needs realtime.

## 4. Screens and widgets

### Routes

```dart
NoteIndexScreen.routeName  // '/note/index'
NoteNewScreen.routeName    // '/note/new'
NoteViewScreen.routeName   // '/note/:id/view'
NoteEditScreen.routeName   // '/note/:id/edit'
```

Router builders for parameterized routes:

```dart
GoRoute(
  path: NoteEditScreen.routeName,
  builder: (context, state) =>
      NoteEditScreen(id: state.pathParameters['id']!),
),
```

Navigate to view/edit with `routeName.replaceFirst(':id', id)`.

### Index

`StatelessWidget`. `Scaffold` + `AppBar` + `BodyContainer(child: NoteList())` + FAB `context.push(NoteNewScreen.routeName)`.

### List

`ConsumerStatefulWidget`:

- Debounce search 300 ms → `ref.read(notesProvider.notifier).setSearchQuery`
- `ref.watch(notesProvider).when(..., skipLoadingOnReload: true)`
- Error: generic `Text('An error occurred. Please try again.')` — do not interpolate `$error`

### List tile

`StatelessWidget`. `context.push(NoteViewScreen.routeName.replaceFirst(':id', note.id))`. No providers.

### New / Edit / View

New: `StatelessWidget` wrapping `NoteForm()`.
Edit: `ConsumerWidget`, `ref.watch(noteProvider(id)).when(...)`, then `NoteForm(note: note)`.
View: `ConsumerWidget`, `ref.watch(noteProvider(id)).when(...)`. FAB → edit. Display the image with `StoredImage(bucket: NoteRepository.imageBucket, path: note.imagePath)`.

### Form — `lib/note/widgets/note_form.dart`

`ConsumerStatefulWidget` with optional `NoteModel? note`. Widgets call the **notifier**, never the repository.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '/core/services/form_validators.dart';
import '/core/widgets/confirmation_dialog.dart';
import '/core/widgets/inputs/image_input.dart';
import '/core/widgets/inputs/text_input.dart';
import '/core/widgets/stored_image.dart';
import '/note/models/note_model.dart';
import '/note/providers/notes_provider.dart';
import '/note/repositories/note_repository.dart';
import '/note/screens/note_index_screen.dart';

class NoteForm extends ConsumerStatefulWidget {
  const NoteForm({super.key, this.note});

  final NoteModel? note;

  bool get isEditForm => note != null;

  @override
  ConsumerState<NoteForm> createState() => _NoteFormState();
}

class _NoteFormState extends ConsumerState<NoteForm> {
  final _formKey = GlobalKey<FormState>();
  final _titleController = TextEditingController();
  final _bodyController = TextEditingController();
  ImageInputValue? _pickedImage;
  bool _removeImage = false;
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    final note = widget.note;
    if (note != null) {
      _titleController.text = note.title;
      _bodyController.text = note.body ?? '';
    }
  }

  @override
  void dispose() {
    _titleController.dispose();
    _bodyController.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);
    try {
      if (widget.isEditForm) {
        await ref.read(notesProvider.notifier).updateNote(
              id: widget.note!.id,
              title: _titleController.text.trim(),
              body: _bodyController.text.trim(),
              imageBytes: _pickedImage?.bytes,
              imageExtension: _pickedImage?.extension ?? 'jpg',
              removeImage: _removeImage,
            );
      } else {
        await ref.read(notesProvider.notifier).createNote(
              title: _titleController.text.trim(),
              body: _bodyController.text.trim(),
              imageBytes: _pickedImage?.bytes,
              imageExtension: _pickedImage?.extension ?? 'jpg',
            );
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

  Future<void> _delete() async {
    await showConfirmationDialog(
      context: context,
      text: 'Delete this note?',
      onConfirm: () {
        _confirmDelete();
      },
    );
  }

  Future<void> _confirmDelete() async {
    setState(() => _isLoading = true);
    try {
      await ref.read(notesProvider.notifier).deleteNote(widget.note!.id);
      if (!mounted) return;
      context.go(NoteIndexScreen.routeName);
    } catch (_) {
      if (!mounted) return;
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('An error occurred. Please try again.')),
      );
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final existingPath =
        _removeImage ? null : widget.note?.imagePath;

    return Form(
      key: _formKey,
      child: Column(
        spacing: 16,
        children: [
          ImageInput(
            value: _pickedImage,
            onChanged: (value) => setState(() {
              _pickedImage = value;
              if (value != null) _removeImage = false;
            }),
            existingImage: existingPath == null
                ? null
                : StoredImage(
                    bucket: NoteRepository.imageBucket,
                    path: existingPath,
                    height: 180,
                  ),
            onExistingImageRemoved: () => setState(() {
              _pickedImage = null;
              _removeImage = true;
            }),
          ),
          TextInput(
            controller: _titleController,
            validator: requiredStringValidator,
            label: 'TITLE',
            textInputAction: TextInputAction.next,
          ),
          TextInput(
            controller: _bodyController,
            label: 'BODY',
            maxLines: 6,
          ),
          ElevatedButton(
            onPressed: _isLoading ? null : _submit,
            child: _isLoading
                ? const CircularProgressIndicator()
                : Text(widget.isEditForm ? 'Save' : 'Create'),
          ),
          if (widget.isEditForm)
            TextButton(
              onPressed: _isLoading ? null : _delete,
              child: const Text('Delete'),
            ),
        ],
      ),
    );
  }
}
```

After every `await`: `if (!mounted) return;`

## 5. Wire-up

1. Add `GoRoute`s in `app_router.dart`
2. If this is the home feature, use its index as `initialLocation` and as the post-login redirect target

## 6. Migration

Put the SQL below in a CLI-generated file. Do not run it only in the Dashboard.

```sh
supabase migration new create_notes_table
# if images:
supabase migration new create_note_images_bucket
```

Then `supabase db reset`. Remote: `supabase db push` when the user is ready. Full CLI: [supabase.md](supabase.md).

```sql
create table public.notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users (id) on delete cascade,
  title text not null,
  body text,
  image_path text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index notes_user_id_idx on public.notes (user_id);

create or replace function public.handle_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger notes_set_updated_at
  before update on public.notes
  for each row
  execute function public.handle_updated_at();

alter table public.notes enable row level security;

create policy "Users can read their own notes"
  on public.notes for select to authenticated
  using (auth.uid() = user_id);

create policy "Users can insert their own notes"
  on public.notes for insert to authenticated
  with check (auth.uid() = user_id);

create policy "Users can update their own notes"
  on public.notes for update to authenticated
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create policy "Users can delete their own notes"
  on public.notes for delete to authenticated
  using (auth.uid() = user_id);
```

Bucket migration (skip if no images):

```sql
insert into storage.buckets (id, name, public)
values ('note-images', 'note-images', false);

create policy "Users can upload their own note images"
  on storage.objects for insert to authenticated
  with check (
    bucket_id = 'note-images'
    and (storage.foldername(name))[1] = auth.uid()::text
  );

create policy "Users can view their own note images"
  on storage.objects for select to authenticated
  using (
    bucket_id = 'note-images'
    and (storage.foldername(name))[1] = auth.uid()::text
  );

create policy "Users can update their own note images"
  on storage.objects for update to authenticated
  using (
    bucket_id = 'note-images'
    and (storage.foldername(name))[1] = auth.uid()::text
  );

create policy "Users can delete their own note images"
  on storage.objects for delete to authenticated
  using (
    bucket_id = 'note-images'
    and (storage.foldername(name))[1] = auth.uid()::text
  );
```

If `handle_updated_at` already exists (later features), omit the function and only add the trigger.

## Repository vs service vs controller vs mapper

| Kind | When | Example |
|---|---|---|
| Repository | One domain table / auth API / `functions.invoke` | `NoteRepository` |
| Service | I/O reused by many features | `ImageStorageService` |
| Edge Function | Secrets, service role, third-party APIs | `supabase/functions/<name>/` |
| Controller | Multi-step UI flow (scan → confirm → result) | `lib/<feature>/controllers/` |
| Mapper | Third-party DTO → domain / form prefill | `lib/<feature>/mappers/` |

Public keyless HTTP APIs get a Dart repository (no Supabase client) plus a mapper. Anything with a secret goes in an Edge Function; the feature repository calls `_client.functions.invoke`. Widgets must not import third-party packages or invoke functions.
