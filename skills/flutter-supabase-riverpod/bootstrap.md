# Bootstrap templates

Use these when scaffolding an app. Replace `<package>`, `<AppTitle>`, and the home screen import with the real names.

Shared widgets, theme, and drawer: [ui.md](ui.md).

## analysis_options.yaml

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  plugins:
    - custom_lint
```

## .env and secrets

`.env` (never commit):

```
SUPABASE_URL=
SUPABASE_PUBLISHABLE_KEY=
```

`.env.example` (commit this):

```
SUPABASE_URL=
SUPABASE_PUBLISHABLE_KEY=
```

`pubspec.yaml` assets:

```yaml
flutter:
  assets:
    - .env
```

Add to `.gitignore` (flutter create does not include these):

```
.env
supabase/.temp/
supabase/.env
```

## lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import 'package:<package>/router/app_router.dart';
import 'package:<package>/themes/light_theme.dart';

Future<void> _initializeSupabase() async {
  await Supabase.initialize(
    url: dotenv.env['SUPABASE_URL']!,
    publishableKey: dotenv.env['SUPABASE_PUBLISHABLE_KEY']!,
  );
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await dotenv.load(fileName: '.env');
  await _initializeSupabase();
  runApp(const ProviderScope(child: App()));
}

class App extends ConsumerWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: '<AppTitle>',
      theme: lightTheme,
      routerConfig: router,
    );
  }
}
```

## lib/core/providers/supabase_provider.dart

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

final supabaseProvider = Provider<SupabaseClient>((ref) {
  return Supabase.instance.client;
});
```

## lib/core/services/image_storage_service.dart

Include this when the app stores images. Persist the returned **path**, not a public URL.

```dart
import 'dart:typed_data';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import '/core/providers/supabase_provider.dart';

final imageStorageServiceProvider = Provider<ImageStorageService>((ref) {
  return ImageStorageService(ref.watch(supabaseProvider));
});

class ImageStorageService {
  ImageStorageService(this._client);

  final SupabaseClient _client;

  Future<String> upload({
    required String bucket,
    required Uint8List imageBytes,
    String imageExtension = 'jpg',
    String? folder,
  }) async {
    final userId = _client.auth.currentUser?.id;
    if (userId == null) {
      throw const AuthException('User must be signed in to upload an image.');
    }

    final extension = imageExtension.replaceAll('.', '');
    final fileName = '${DateTime.now().millisecondsSinceEpoch}.$extension';
    final path =
        folder != null ? '$userId/$folder/$fileName' : '$userId/$fileName';

    await _client.storage.from(bucket).uploadBinary(
          path,
          imageBytes,
          fileOptions: FileOptions(
            contentType: _contentTypeForExtension(extension),
            upsert: true,
          ),
        );

    return path;
  }

  Future<String?> signedUrl({
    required String bucket,
    required String? path,
    int expiresIn = 3600,
  }) async {
    if (path == null || path.isEmpty) return null;
    return _client.storage.from(bucket).createSignedUrl(path, expiresIn);
  }

  Future<Uint8List> download({
    required String bucket,
    required String path,
  }) {
    return _client.storage.from(bucket).download(path);
  }

  Future<String> copy({
    required String bucket,
    required String path,
  }) async {
    final bytes = await download(bucket: bucket, path: path);
    return upload(
      bucket: bucket,
      imageBytes: bytes,
      imageExtension: path.split('.').last,
    );
  }

  Future<void> delete({required String bucket, required String path}) {
    return _client.storage.from(bucket).remove([path]);
  }

  String _contentTypeForExtension(String extension) {
    return switch (extension.toLowerCase()) {
      'png' => 'image/png',
      'webp' => 'image/webp',
      'gif' => 'image/gif',
      _ => 'image/jpeg',
    };
  }
}
```

## lib/auth/repositories/auth_repository.dart

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import '/core/providers/supabase_provider.dart';

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepository(ref.watch(supabaseProvider));
});

class AuthRepository {
  AuthRepository(this._client);

  final SupabaseClient _client;

  GoTrueClient get _auth => _client.auth;

  Session? get currentSession => _auth.currentSession;

  Stream<Session?> get authStateChanges =>
      _auth.onAuthStateChange.map((data) => data.session);

  Future<bool> validateSession() async {
    if (_auth.currentSession == null) return false;
    try {
      await _auth.getUser();
      return true;
    } catch (_) {
      await _auth.signOut();
      return false;
    }
  }

  Future<Session?> signUp({
    required String email,
    required String password,
  }) async {
    final response = await _auth.signUp(email: email, password: password);
    return response.session;
  }

  Future<void> signIn({required String email, required String password}) {
    return _auth.signInWithPassword(email: email, password: password);
  }

  Future<void> signOut() => _auth.signOut();
}
```

## lib/auth/providers/auth_provider.dart

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

import '/auth/repositories/auth_repository.dart';

final authProvider = StreamNotifierProvider<AuthNotifier, Session?>(() {
  return AuthNotifier();
});

class AuthNotifier extends StreamNotifier<Session?> {
  @override
  Stream<Session?> build() async* {
    final repo = ref.watch(authRepositoryProvider);
    await repo.validateSession();
    yield repo.currentSession;
    yield* repo.authStateChanges;
  }

  Future<Session?> signUp(String email, String password) =>
      ref.read(authRepositoryProvider).signUp(email: email, password: password);

  Future<void> signIn(String email, String password) =>
      ref.read(authRepositoryProvider).signIn(email: email, password: password);

  Future<void> signOut() => ref.read(authRepositoryProvider).signOut();
}
```

Mutations only call the repository. The stream updates the UI and the router.

## lib/router/app_router.dart

Replace `HomeIndexScreen` with the first feature index. Keep redirect order.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '/auth/providers/auth_provider.dart';
import '/auth/screens/login_screen.dart';
import '/auth/screens/signup_screen.dart';
import '/core/screens/splash_screen.dart';
// import feature screens…

final routerProvider = Provider<GoRouter>((ref) {
  final refreshNotifier = _AuthRefreshNotifier(ref);
  ref.onDispose(refreshNotifier.dispose);

  return GoRouter(
    initialLocation: HomeIndexScreen.routeName,
    refreshListenable: refreshNotifier,
    redirect: (context, state) {
      final authState = ref.read(authProvider);
      final isLoggedIn = authState.value != null;
      final isAuthRoute = state.matchedLocation.startsWith('/auth');

      if (authState.isLoading) {
        return SplashScreen.routeName;
      }

      if (state.matchedLocation == SplashScreen.routeName) {
        return isLoggedIn ? HomeIndexScreen.routeName : LoginScreen.routeName;
      }

      if (!isLoggedIn && !isAuthRoute) {
        return LoginScreen.routeName;
      }

      if (isLoggedIn && isAuthRoute) {
        return HomeIndexScreen.routeName;
      }

      return null;
    },
    routes: [
      GoRoute(
        path: SplashScreen.routeName,
        builder: (context, state) => const SplashScreen(),
      ),
      GoRoute(
        path: LoginScreen.routeName,
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: SignupScreen.routeName,
        builder: (context, state) => const SignupScreen(),
      ),
      // feature routes…
    ],
  );
});

class _AuthRefreshNotifier extends ChangeNotifier {
  _AuthRefreshNotifier(this._ref) {
    _ref.listen(authProvider, (_, _) => notifyListeners());
  }

  final Ref _ref;
}
```

## lib/core/screens/splash_screen.dart

```dart
import 'package:flutter/material.dart';

class SplashScreen extends StatelessWidget {
  static const routeName = '/splash';

  const SplashScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(body: Center(child: CircularProgressIndicator()));
  }
}
```

No providers. No business logic.

## Auth screens

- Routes: `/auth/login`, `/auth/signup`
- `StatelessWidget` + `BodyContainer` + form + text button to the other auth route (`context.go`)
- Forms are `ConsumerStatefulWidget`
- Local `_isLoading`; `ref.read(authProvider.notifier).signIn/signUp`
- Do not navigate after login — the router redirect does it
- After signup: if `session == null`, show “Check your inbox to confirm your email.” (Confirm email is on by default in Supabase.) Do not `context.go`.

### lib/core/services/form_validators.dart

```dart
String? emailValidator(String? value) {
  if (value == null || value.isEmpty) return 'Email is required.';
  final emailRegex = RegExp(r'^[^@]+@[^@]+\.[^@]+');
  if (!emailRegex.hasMatch(value)) return 'Enter a valid email address.';
  return null;
}

String? requiredStringValidator(String? value) {
  if (value == null || value.trim().isEmpty) return 'This field is required.';
  return null;
}

String? passwordValidator(String? value) {
  if (value == null || value.isEmpty) return 'Password is required.';
  if (value.length < 6) return 'Password must be at least 6 characters.';
  return null;
}

String? confirmPasswordValidator(String? value, String password) {
  if (value == null || value.isEmpty) return 'Confirm your password.';
  if (value != password) return 'Passwords do not match.';
  return null;
}
```

### lib/auth/screens/login_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

import '/auth/screens/signup_screen.dart';
import '/auth/widgets/login_form.dart';
import '/core/widgets/body_container.dart';

class LoginScreen extends StatelessWidget {
  static const routeName = '/auth/login';

  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: BodyContainer(
        child: SingleChildScrollView(
          child: Column(
            spacing: 32,
            children: [
              const LoginForm(),
              TextButton(
                onPressed: () => context.go(SignupScreen.routeName),
                child: const Text("Don't have an account? Sign up"),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

Signup screen is the same shape: `routeName = '/auth/signup'`, `SignupForm`, button `context.go(LoginScreen.routeName)`.

### lib/auth/widgets/login_form.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '/auth/providers/auth_provider.dart';
import '/core/services/form_validators.dart';
import '/core/widgets/inputs/text_input.dart';

class LoginForm extends ConsumerStatefulWidget {
  const LoginForm({super.key});

  @override
  ConsumerState<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends ConsumerState<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isLoading = false;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _login() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);
    try {
      await ref.read(authProvider.notifier).signIn(
            _emailController.text.trim(),
            _passwordController.text,
          );
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
    return Form(
      key: _formKey,
      child: Column(
        spacing: 16,
        children: [
          TextInput(
            controller: _emailController,
            validator: emailValidator,
            keyboardType: TextInputType.emailAddress,
            textInputAction: TextInputAction.next,
            autocorrect: false,
            label: 'EMAIL',
            hintText: 'e.g. john@example.com',
          ),
          TextInput(
            controller: _passwordController,
            validator: requiredStringValidator,
            obscureText: true,
            enableSuggestions: false,
            autocorrect: false,
            textInputAction: TextInputAction.done,
            label: 'PASSWORD',
            hintText: 'Enter your password',
          ),
          ElevatedButton(
            onPressed: _isLoading ? null : _login,
            child: _isLoading
                ? const CircularProgressIndicator()
                : const Text('Login'),
          ),
        ],
      ),
    );
  }
}
```

### lib/auth/widgets/signup_form.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '/auth/providers/auth_provider.dart';
import '/core/services/form_validators.dart';
import '/core/widgets/inputs/text_input.dart';

class SignupForm extends ConsumerStatefulWidget {
  const SignupForm({super.key});

  @override
  ConsumerState<SignupForm> createState() => _SignupFormState();
}

class _SignupFormState extends ConsumerState<SignupForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _confirmPasswordController = TextEditingController();
  bool _isLoading = false;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    _confirmPasswordController.dispose();
    super.dispose();
  }

  Future<void> _signUp() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);
    try {
      final session = await ref.read(authProvider.notifier).signUp(
            _emailController.text.trim(),
            _passwordController.text,
          );
      if (!mounted) return;
      if (session == null) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Check your inbox to confirm your email.'),
          ),
        );
      }
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
    return Form(
      key: _formKey,
      child: Column(
        spacing: 16,
        children: [
          TextInput(
            controller: _emailController,
            validator: emailValidator,
            keyboardType: TextInputType.emailAddress,
            textInputAction: TextInputAction.next,
            autocorrect: false,
            label: 'EMAIL',
            hintText: 'e.g. john@example.com',
          ),
          TextInput(
            controller: _passwordController,
            validator: passwordValidator,
            obscureText: true,
            enableSuggestions: false,
            autocorrect: false,
            textInputAction: TextInputAction.next,
            label: 'PASSWORD',
            hintText: 'At least 6 characters',
          ),
          TextInput(
            controller: _confirmPasswordController,
            validator: (value) =>
                confirmPasswordValidator(value, _passwordController.text),
            obscureText: true,
            enableSuggestions: false,
            autocorrect: false,
            textInputAction: TextInputAction.done,
            label: 'CONFIRM PASSWORD',
            hintText: 'Re-enter your password',
          ),
          ElevatedButton(
            onPressed: _isLoading ? null : _signUp,
            child: _isLoading
                ? const CircularProgressIndicator()
                : const Text('Sign up'),
          ),
        ],
      ),
    );
  }
}
```
