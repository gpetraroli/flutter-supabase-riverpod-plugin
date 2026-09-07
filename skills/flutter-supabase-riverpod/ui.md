# Shared UI templates

Write these into every app. Adapt labels and colors; keep the structure.

## lib/core/widgets/body_container.dart

```dart
import 'package:flutter/material.dart';

class BodyContainer extends StatelessWidget {
  const BodyContainer({
    super.key,
    required this.child,
    this.padding = defaultPadding,
  });

  final Widget child;
  final double padding;

  static const double defaultPadding = 16;

  @override
  Widget build(BuildContext context) {
    return SafeArea(
      top: false,
      child: Padding(padding: EdgeInsets.all(padding), child: child),
    );
  }
}
```

## lib/core/widgets/inputs/text_input.dart

```dart
import 'package:flutter/material.dart';

class TextInput extends StatelessWidget {
  const TextInput({
    super.key,
    required this.controller,
    this.validator,
    this.label,
    this.hintText,
    this.textInputAction,
    this.keyboardType,
    this.maxLines,
    this.onChanged,
    this.obscureText = false,
    this.enableSuggestions = true,
    this.autocorrect = true,
  });

  final TextEditingController controller;
  final String? Function(String?)? validator;
  final String? label;
  final String? hintText;
  final TextInputAction? textInputAction;
  final TextInputType? keyboardType;
  final int? maxLines;
  final ValueChanged<String>? onChanged;
  final bool obscureText;
  final bool enableSuggestions;
  final bool autocorrect;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      spacing: 8,
      children: [
        if (label != null)
          Text(label!, style: Theme.of(context).textTheme.labelLarge),
        TextFormField(
          controller: controller,
          validator: validator,
          textInputAction: textInputAction,
          keyboardType: keyboardType,
          maxLines: obscureText ? 1 : maxLines,
          obscureText: obscureText,
          enableSuggestions: enableSuggestions,
          autocorrect: autocorrect,
          onChanged: onChanged,
          decoration: InputDecoration(hintText: hintText),
        ),
      ],
    );
  }
}
```

## lib/core/widgets/inputs/image_input.dart

Add `image_picker` when a feature stores images. Keep picked bytes in the form until save; the repository uploads.

```dart
import 'dart:typed_data';

import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';

class ImageInputValue {
  const ImageInputValue({required this.bytes, required this.extension});

  final Uint8List bytes;
  final String extension;
}

class ImageInput extends StatefulWidget {
  const ImageInput({
    super.key,
    required this.onChanged,
    this.value,
    this.existingImage,
    this.onExistingImageRemoved,
    this.placeholderText = 'ADD A PHOTO',
    this.height = 180,
  });

  final ValueChanged<ImageInputValue?> onChanged;
  final ImageInputValue? value;
  final Widget? existingImage;
  final VoidCallback? onExistingImageRemoved;
  final String placeholderText;
  final double height;

  @override
  State<ImageInput> createState() => _ImageInputState();
}

enum _ImageInputAction { camera, gallery, remove }

class _ImageInputState extends State<ImageInput> {
  static const _maxImageWidth = 1280.0;
  static const _imageQuality = 80;

  final _imagePicker = ImagePicker();

  Future<void> _pickImage(ImageSource source) async {
    final XFile? file;
    try {
      file = await _imagePicker.pickImage(
        source: source,
        maxWidth: _maxImageWidth,
        imageQuality: _imageQuality,
      );
    } on Exception {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Could not pick an image.')),
        );
      }
      return;
    }

    if (file == null) return;

    final bytes = await file.readAsBytes();
    if (!mounted) return;

    final nameParts = file.name.split('.');
    var extension =
        nameParts.length > 1 ? nameParts.last.toLowerCase() : 'jpg';
    if (extension == 'jpeg') extension = 'jpg';

    widget.onChanged(ImageInputValue(bytes: bytes, extension: extension));
  }

  Future<void> _showSourcePicker() async {
    final action = await showModalBottomSheet<_ImageInputAction>(
      context: context,
      builder: (context) => SafeArea(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ListTile(
              leading: const Icon(Icons.camera_alt_outlined),
              title: const Text('Take photo'),
              onTap: () => Navigator.pop(context, _ImageInputAction.camera),
            ),
            ListTile(
              leading: const Icon(Icons.photo_library_outlined),
              title: const Text('Choose from gallery'),
              onTap: () => Navigator.pop(context, _ImageInputAction.gallery),
            ),
            if (widget.value != null || widget.existingImage != null)
              ListTile(
                leading: const Icon(Icons.delete_outline),
                title: const Text('Remove photo'),
                onTap: () => Navigator.pop(context, _ImageInputAction.remove),
              ),
          ],
        ),
      ),
    );

    if (action == null || !mounted) return;

    switch (action) {
      case _ImageInputAction.camera:
        await _pickImage(ImageSource.camera);
      case _ImageInputAction.gallery:
        await _pickImage(ImageSource.gallery);
      case _ImageInputAction.remove:
        if (widget.value != null) {
          widget.onChanged(null);
        } else {
          widget.onExistingImageRemoved?.call();
        }
    }
  }

  @override
  Widget build(BuildContext context) {
    final scheme = Theme.of(context).colorScheme;

    return GestureDetector(
      onTap: _showSourcePicker,
      child: Container(
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(16),
          border: Border.all(color: scheme.outline),
        ),
        clipBehavior: Clip.antiAlias,
        height: widget.height,
        width: double.infinity,
        child: widget.value != null
            ? Image.memory(
                widget.value!.bytes,
                fit: BoxFit.cover,
                width: double.infinity,
                height: widget.height,
                gaplessPlayback: true,
              )
            : widget.existingImage != null
                ? widget.existingImage!
                : Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    spacing: 8,
                    children: [
                      Icon(Icons.add_a_photo_outlined, color: scheme.outline),
                      Text(
                        widget.placeholderText,
                        style: Theme.of(context).textTheme.labelMedium,
                      ),
                    ],
                  ),
      ),
    );
  }
}
```

## lib/core/widgets/confirmation_dialog.dart

```dart
import 'package:flutter/material.dart';

Future<void> showConfirmationDialog({
  required BuildContext context,
  required String text,
  required VoidCallback onConfirm,
}) {
  return showDialog<void>(
    context: context,
    builder: (context) => AlertDialog(
      content: Text(text),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: const Text('Cancel'),
        ),
        TextButton(
          onPressed: () {
            Navigator.of(context).pop();
            onConfirm();
          },
          child: const Text('Confirm'),
        ),
      ],
    ),
  );
}
```

## lib/core/widgets/information_dialog.dart

```dart
import 'package:flutter/material.dart';

Future<void> showInformationDialog({
  required BuildContext context,
  required String text,
  List<String> items = const [],
}) {
  return showDialog<void>(
    context: context,
    builder: (context) => AlertDialog(
      content: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(text),
          for (final item in items) Text('• $item'),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: const Text('OK'),
        ),
      ],
    ),
  );
}
```

Use `showInformationDialog` when delete is blocked because other rows still reference the entity.

## lib/core/widgets/stored_image.dart

`path` is a storage path, not a public URL.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '/core/services/image_storage_service.dart';

class StoredImage extends ConsumerStatefulWidget {
  const StoredImage({
    super.key,
    required this.bucket,
    this.path,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
  });

  final String bucket;
  final String? path;
  final double? width;
  final double? height;
  final BoxFit fit;

  @override
  ConsumerState<StoredImage> createState() => _StoredImageState();
}

class _StoredImageState extends ConsumerState<StoredImage> {
  String? _url;

  @override
  void initState() {
    super.initState();
    _loadUrl();
  }

  @override
  void didUpdateWidget(covariant StoredImage oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.path != widget.path || oldWidget.bucket != widget.bucket) {
      _loadUrl();
    }
  }

  Future<void> _loadUrl() async {
    final path = widget.path;
    if (path == null || path.isEmpty) {
      if (mounted) setState(() => _url = null);
      return;
    }

    final url = await ref
        .read(imageStorageServiceProvider)
        .signedUrl(bucket: widget.bucket, path: path);

    if (mounted) setState(() => _url = url);
  }

  @override
  Widget build(BuildContext context) {
    if (_url != null) {
      return Image.network(
        _url!,
        width: widget.width,
        height: widget.height,
        fit: widget.fit,
      );
    }

    return Container(
      width: widget.width,
      height: widget.height,
      color: Theme.of(context).colorScheme.surfaceContainerHighest,
      child: const Icon(Icons.image_outlined),
    );
  }
}
```

## Theme

Put `ThemeData` in `lib/themes/light_theme.dart`. `App` uses `theme: lightTheme`. Do not scatter `ThemeData` in widgets.

```dart
import 'package:flutter/material.dart';

const _seed = Color(0xFF2E7D32);

final lightTheme = ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(seedColor: _seed),
  elevatedButtonTheme: ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      minimumSize: const Size(double.infinity, 48),
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
    ),
  ),
);
```

Change the seed to match the app.

## Drawer

### lib/navigation/widgets/drawer_list_tile.dart

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class DrawerListTile extends StatelessWidget {
  const DrawerListTile({
    super.key,
    required this.title,
    required this.route,
    required this.icon,
  });

  final String title;
  final String route;
  final IconData icon;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(title),
      leading: Icon(icon),
      onTap: () => context.go(route),
    );
  }
}
```

### lib/navigation/widgets/drawer_menu.dart

`ConsumerWidget`. Feature tiles use `DrawerListTile`. Logout: `ref.read(authProvider.notifier).signOut()` — no manual navigation.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '/auth/providers/auth_provider.dart';
import '/navigation/widgets/drawer_list_tile.dart';
// import feature index screens…

class DrawerMenu extends ConsumerWidget {
  const DrawerMenu({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Drawer(
      child: SafeArea(
        child: Column(
          children: [
            Text('<AppTitle>', style: Theme.of(context).textTheme.titleLarge),
            const SizedBox(height: 32),
            Expanded(
              child: ListView(
                children: [
                  DrawerListTile(
                    title: 'Notes',
                    route: NoteIndexScreen.routeName,
                    icon: Icons.notes,
                  ),
                ],
              ),
            ),
            TextButton(
              onPressed: () => ref.read(authProvider.notifier).signOut(),
              child: const Text('Logout'),
            ),
          ],
        ),
      ),
    );
  }
}
```
