# Security Patterns — OAuth2 PKCE (Keycloak via flutter_appauth) and Local JWT

This reference describes the complete authentication architecture for the spec. Include
all of this content in Section 6 of the generated specification.

---

## Architecture Overview — Keycloak PKCE (Auth = Keycloak)

The Flutter app operates as an OAuth2 Public Client using Authorization Code flow with
PKCE (Proof Key for Code Exchange) and OIDC, via the `flutter_appauth` package which
wraps the native AppAuth libraries (Android + iOS). Keycloak is the external identity
provider that handles login UI, user registration, and role management.

Responsibilities:

1. Open the system browser (Chrome Custom Tabs / SFAuthenticationSession) for login
2. Handle the OAuth2 callback via a registered custom URL scheme
3. Exchange the authorization code for ID token + access token + refresh token (PKCE)
4. Store tokens securely in `flutter_secure_storage` (Keychain on iOS, EncryptedSharedPreferences on Android)
5. Extract roles from OIDC ID token claims (`realm_access.roles`, `resource_access.<client>.roles`)
6. Attach Bearer token to all Dio requests via interceptor
7. Refresh tokens silently in the background or on 401 responses

The flow:
```
User navigates to protected route
    → go_router redirect detects unauthenticated state
    → AuthRepository.signIn() invokes FlutterAppAuth.authorizeAndExchangeCode(...)
    → System browser opens Keycloak /auth?response_type=code&code_challenge=...
    → User logs in at Keycloak
    → Keycloak redirects to {{APPLICATION_ID}}://oauth2redirect?code=...
    → Native AppAuth handler completes PKCE exchange (ID + access + refresh tokens)
    → Tokens persisted to flutter_secure_storage
    → AuthNotifier emits AuthState.authenticated(user, accessToken)
    → go_router rebuilds, user lands on the protected route
    → Dio interceptor reads token from secure storage and attaches Bearer
```

---

## URL Scheme Registration

### Android (`android/app/src/main/AndroidManifest.xml`)

Inside the `<activity>` element for `MainActivity`:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="{{APPLICATION_ID}}"/>
</intent-filter>
```

Also add the AppAuth redirect scheme placeholder to `android/app/build.gradle`:

```groovy
android {
  defaultConfig {
    manifestPlaceholders = [
      'appAuthRedirectScheme': '{{APPLICATION_ID}}'
    ]
  }
}
```

### iOS (`ios/Runner/Info.plist`)

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>{{APPLICATION_ID}}</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>{{APPLICATION_ID}}</string>
    </array>
  </dict>
</array>
```

---

## AuthRepository (Keycloak)

```dart
// lib/core/auth/auth_repository.dart
import 'package:flutter_appauth/flutter_appauth.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import 'auth_state.dart';
import 'jwt_decoder.dart';

part 'auth_repository.g.dart';

class AuthRepository {
  AuthRepository(this._secureStorage) : _appAuth = const FlutterAppAuth();

  final FlutterSecureStorage _secureStorage;
  final FlutterAppAuth _appAuth;

  static const _accessTokenKey = 'auth_access_token';
  static const _refreshTokenKey = 'auth_refresh_token';
  static const _idTokenKey = 'auth_id_token';

  String get _issuer => dotenv.env['KEYCLOAK_ISSUER']!;
  String get _clientId => dotenv.env['KEYCLOAK_CLIENT_ID']!;
  String get _redirectUri => dotenv.env['KEYCLOAK_REDIRECT_URI']!;
  String get _postLogoutRedirectUri => dotenv.env['KEYCLOAK_POST_LOGOUT_REDIRECT_URI']!;

  /// Triggers the system browser flow and returns the authenticated state.
  Future<AuthState> signIn() async {
    final result = await _appAuth.authorizeAndExchangeCode(
      AuthorizationTokenRequest(
        _clientId,
        _redirectUri,
        issuer: _issuer,
        scopes: const ['openid', 'profile', 'email', 'offline_access'],
        promptValues: const ['login'],
      ),
    );

    await _persistTokens(result);
    return _stateFromTokens(result.accessToken!, result.idToken!);
  }

  /// Restores session at app startup, refreshing the token if needed.
  Future<AuthState?> restoreSession() async {
    final accessToken = await _secureStorage.read(key: _accessTokenKey);
    final refreshToken = await _secureStorage.read(key: _refreshTokenKey);
    final idToken = await _secureStorage.read(key: _idTokenKey);

    if (refreshToken == null) return null;

    // If access token is still valid, use it directly
    if (accessToken != null && idToken != null && !JwtDecoder.isExpired(accessToken)) {
      return _stateFromTokens(accessToken, idToken);
    }

    // Otherwise refresh
    try {
      final result = await _appAuth.token(
        TokenRequest(
          _clientId,
          _redirectUri,
          issuer: _issuer,
          refreshToken: refreshToken,
          scopes: const ['openid', 'profile', 'email', 'offline_access'],
        ),
      );
      await _persistTokens(result);
      return _stateFromTokens(result.accessToken!, result.idToken!);
    } catch (_) {
      await signOutLocal();
      return null;
    }
  }

  Future<String?> getAccessToken() async {
    final token = await _secureStorage.read(key: _accessTokenKey);
    if (token == null) return null;
    if (JwtDecoder.isExpired(token)) {
      final restored = await restoreSession();
      if (restored is _Authenticated) return restored.accessToken;
      return null;
    }
    return token;
  }

  Future<void> signOut() async {
    final idToken = await _secureStorage.read(key: _idTokenKey);
    if (idToken != null) {
      await _appAuth.endSession(
        EndSessionRequest(
          idTokenHint: idToken,
          postLogoutRedirectUrl: _postLogoutRedirectUri,
          issuer: _issuer,
        ),
      );
    }
    await signOutLocal();
  }

  Future<void> signOutLocal() async {
    await _secureStorage.delete(key: _accessTokenKey);
    await _secureStorage.delete(key: _refreshTokenKey);
    await _secureStorage.delete(key: _idTokenKey);
  }

  Future<void> _persistTokens(TokenResponse result) async {
    await _secureStorage.write(key: _accessTokenKey, value: result.accessToken);
    if (result.refreshToken != null) {
      await _secureStorage.write(key: _refreshTokenKey, value: result.refreshToken);
    }
    if (result.idToken != null) {
      await _secureStorage.write(key: _idTokenKey, value: result.idToken);
    }
  }

  AuthState _stateFromTokens(String accessToken, String idToken) {
    final claims = JwtDecoder.decode(idToken);
    final user = AuthUser(
      id: claims['sub'] as String? ?? '',
      username: claims['preferred_username'] as String? ?? '',
      email: claims['email'] as String? ?? '',
      name: claims['name'] as String? ?? '',
      roles: _extractRoles(claims),
    );
    return AuthState.authenticated(user: user, accessToken: accessToken);
  }

  List<String> _extractRoles(Map<String, dynamic> claims) {
    final roles = <String>{};
    final realmAccess = claims['realm_access'] as Map<String, dynamic>?;
    if (realmAccess != null) {
      final realmRoles = (realmAccess['roles'] as List?)?.cast<String>() ?? const [];
      roles.addAll(realmRoles);
    }
    final resourceAccess = claims['resource_access'] as Map<String, dynamic>?;
    if (resourceAccess != null) {
      final clientAccess = resourceAccess[_clientId] as Map<String, dynamic>?;
      if (clientAccess != null) {
        final clientRoles = (clientAccess['roles'] as List?)?.cast<String>() ?? const [];
        roles.addAll(clientRoles);
      }
    }
    return roles.map((r) => r.toUpperCase()).toList();
  }
}

@Riverpod(keepAlive: true)
AuthRepository authRepository(AuthRepositoryRef ref) {
  return AuthRepository(const FlutterSecureStorage());
}
```

### Lightweight JWT Decoder

```dart
// lib/core/auth/jwt_decoder.dart
import 'dart:convert';

class JwtDecoder {
  JwtDecoder._();

  static Map<String, dynamic> decode(String token) {
    final parts = token.split('.');
    if (parts.length != 3) throw const FormatException('Invalid JWT');
    final payload = parts[1];
    final normalised = base64Url.normalize(payload);
    final decoded = utf8.decode(base64Url.decode(normalised));
    return json.decode(decoded) as Map<String, dynamic>;
  }

  static bool isExpired(String token) {
    final claims = decode(token);
    final exp = claims['exp'];
    if (exp is! int) return true;
    final expiry = DateTime.fromMillisecondsSinceEpoch(exp * 1000);
    // 30-second skew so we don't ride right to the wire
    return DateTime.now().isAfter(expiry.subtract(const Duration(seconds: 30)));
  }
}
```

---

## AuthState (freezed)

```dart
// lib/core/auth/auth_state.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_state.freezed.dart';

@freezed
class AuthUser with _$AuthUser {
  const factory AuthUser({
    required String id,
    required String username,
    required String email,
    required String name,
    required List<String> roles,
  }) = _AuthUser;

  const AuthUser._();

  bool hasRole(String role) => roles.contains(role.toUpperCase());
  bool hasAnyRole(List<String> required) =>
      required.any((r) => roles.contains(r.toUpperCase()));
}

@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.unauthenticated() = _Unauthenticated;
  const factory AuthState.authenticated({
    required AuthUser user,
    required String accessToken,
  }) = _Authenticated;
}
```

---

## AuthProvider (Riverpod)

```dart
// lib/core/auth/auth_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import 'auth_repository.dart';
import 'auth_state.dart';

part 'auth_provider.g.dart';

@Riverpod(keepAlive: true)
class Auth extends _$Auth {
  @override
  FutureOr<AuthState> build() async {
    final repo = ref.watch(authRepositoryProvider);
    final restored = await repo.restoreSession();
    return restored ?? const AuthState.unauthenticated();
  }

  /// Triggers the Keycloak system-browser sign-in flow.
  Future<void> signIn() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(authRepositoryProvider);
      return repo.signIn();
    });
  }

  Future<void> signOut() async {
    final repo = ref.read(authRepositoryProvider);
    await repo.signOut();
    state = const AsyncData(AuthState.unauthenticated());
  }
}
```

---

## Role Constants

```dart
// lib/core/auth/roles.dart
class Roles {
  Roles._();

  // Replace with actual roles from mockup role folders / Keycloak realm.
  static const String admin = 'ADMIN';
  static const String worker = 'WORKER';
  static const String user = 'USER';
}
```

---

## Auth Interceptor (Dio)

```dart
// lib/core/api/auth_interceptor.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../auth/auth_provider.dart';
import '../auth/auth_repository.dart';
import '../auth/auth_state.dart';

class AuthInterceptor extends Interceptor {
  AuthInterceptor(this._ref);
  final Ref _ref;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final repo = _ref.read(authRepositoryProvider);
    final token = await repo.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Force refresh + retry once
      final repo = _ref.read(authRepositoryProvider);
      final restored = await repo.restoreSession();
      if (restored is _Authenticated) {
        final requestOptions = err.requestOptions;
        requestOptions.headers['Authorization'] = 'Bearer ${restored.accessToken}';
        try {
          final cloneReq = await Dio().fetch<dynamic>(requestOptions);
          return handler.resolve(cloneReq);
        } catch (e) {
          // fall through to original error
        }
      } else {
        // Refresh failed — force logout
        await _ref.read(authProvider.notifier).signOut();
      }
    }
    handler.next(err);
  }
}
```

---

## ID Token Structure Reference

Keycloak places roles in the ID token as follows:

```json
{
  "sub": "f1234-abcd-5678",
  "preferred_username": "john.doe",
  "email": "john.doe@example.com",
  "name": "John Doe",
  "realm_access": {
    "roles": ["USER", "offline_access"]
  },
  "resource_access": {
    "{{KEYCLOAK_CLIENT_ID}}": {
      "roles": ["ADMIN"]
    }
  },
  "scope": "openid profile email offline_access"
}
```

This structure is what `_extractRoles()` parses.

---

# Architecture Overview — Local JWT Auth (Auth = Local)

When the backend manages authentication (no external IdP), the flow is:

```
User submits login form
    → AuthRepository.login(email, password)
    → POST /auth/login → { accessToken, refreshToken, user }
    → Tokens persisted to flutter_secure_storage
    → AuthNotifier emits AuthState.authenticated(user, accessToken)
    → Dio interceptor attaches Bearer header
    → On 401: interceptor calls /auth/refresh with refreshToken cookie/body
```

## AuthRepository (Local JWT)

```dart
// lib/core/auth/auth_repository.dart  (Local JWT variant)
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../api/dio_provider.dart';
import 'auth_state.dart';
import 'jwt_decoder.dart';

part 'auth_repository.g.dart';

class AuthRepository {
  AuthRepository(this._dio, this._secureStorage);
  final Dio _dio;
  final FlutterSecureStorage _secureStorage;

  static const _accessTokenKey = 'auth_access_token';
  static const _refreshTokenKey = 'auth_refresh_token';
  static const _userKey = 'auth_user';

  Future<AuthState> login(String email, String password) async {
    final response = await _dio.post<Map<String, dynamic>>(
      '/auth/login',
      data: {'email': email, 'password': password},
    );
    final body = response.data!;
    final accessToken = body['accessToken'] as String;
    final refreshToken = body['refreshToken'] as String;
    final userJson = body['user'] as Map<String, dynamic>;

    await _secureStorage.write(key: _accessTokenKey, value: accessToken);
    await _secureStorage.write(key: _refreshTokenKey, value: refreshToken);
    await _secureStorage.write(key: _userKey, value: json.encode(userJson));

    final user = AuthUser.fromJson(userJson);
    return AuthState.authenticated(user: user, accessToken: accessToken);
  }

  Future<AuthState?> restoreSession() async {
    final accessToken = await _secureStorage.read(key: _accessTokenKey);
    final userJsonString = await _secureStorage.read(key: _userKey);
    if (accessToken == null || userJsonString == null) return null;

    if (JwtDecoder.isExpired(accessToken)) {
      return _refreshSession();
    }

    final user = AuthUser.fromJson(json.decode(userJsonString));
    return AuthState.authenticated(user: user, accessToken: accessToken);
  }

  Future<AuthState?> _refreshSession() async {
    final refreshToken = await _secureStorage.read(key: _refreshTokenKey);
    if (refreshToken == null) return null;
    try {
      final response = await _dio.post<Map<String, dynamic>>(
        '/auth/refresh',
        data: {'refreshToken': refreshToken},
      );
      final body = response.data!;
      final newAccess = body['accessToken'] as String;
      final newRefresh = body['refreshToken'] as String?;
      await _secureStorage.write(key: _accessTokenKey, value: newAccess);
      if (newRefresh != null) {
        await _secureStorage.write(key: _refreshTokenKey, value: newRefresh);
      }
      final userJsonString = await _secureStorage.read(key: _userKey);
      final user = AuthUser.fromJson(json.decode(userJsonString!));
      return AuthState.authenticated(user: user, accessToken: newAccess);
    } catch (_) {
      await logoutLocal();
      return null;
    }
  }

  Future<String?> getAccessToken() async {
    final token = await _secureStorage.read(key: _accessTokenKey);
    if (token == null) return null;
    if (JwtDecoder.isExpired(token)) {
      final refreshed = await _refreshSession();
      if (refreshed is _Authenticated) return refreshed.accessToken;
      return null;
    }
    return token;
  }

  Future<void> logout() async {
    try {
      await _dio.post<void>('/auth/logout');
    } catch (_) {
      // swallow — proceed with local cleanup
    }
    await logoutLocal();
  }

  Future<void> logoutLocal() async {
    await _secureStorage.delete(key: _accessTokenKey);
    await _secureStorage.delete(key: _refreshTokenKey);
    await _secureStorage.delete(key: _userKey);
  }
}

@Riverpod(keepAlive: true)
AuthRepository authRepository(AuthRepositoryRef ref) {
  return AuthRepository(
    ref.watch(dioProvider),
    const FlutterSecureStorage(),
  );
}
```

## Login Screen (Local JWT)

```dart
// lib/features/auth/presentation/login_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

import '../../../core/auth/auth_provider.dart';
import '../../../core/utils/validators.dart';
import '../../../shared/form/form_text_field.dart';

class LoginScreen extends HookConsumerWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formKey = useMemoized(GlobalKey<FormState>.new);
    final emailCtrl = useTextEditingController();
    final passCtrl = useTextEditingController();
    final loading = useState(false);
    final errorMessage = useState<String?>(null);

    Future<void> submit() async {
      if (!formKey.currentState!.validate()) return;
      loading.value = true;
      errorMessage.value = null;
      try {
        await ref.read(authProvider.notifier).login(emailCtrl.text, passCtrl.text);
      } catch (e) {
        errorMessage.value = 'Invalid email or password';
      } finally {
        loading.value = false;
      }
    }

    return Scaffold(
      body: SafeArea(
        child: Center(
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Form(
              key: formKey,
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(
                    'Sign in',
                    style: Theme.of(context).textTheme.displaySmall,
                  ),
                  const SizedBox(height: 24),
                  FormTextField(
                    controller: emailCtrl,
                    label: 'Email',
                    keyboardType: TextInputType.emailAddress,
                    validator: Validators.compose([
                      (v) => Validators.required(v, label: 'Email'),
                      Validators.email,
                    ]),
                  ),
                  const SizedBox(height: 16),
                  FormTextField(
                    controller: passCtrl,
                    label: 'Password',
                    obscureText: true,
                    validator: (v) => Validators.required(v, label: 'Password'),
                  ),
                  if (errorMessage.value != null) ...[
                    const SizedBox(height: 16),
                    Text(
                      errorMessage.value!,
                      style: TextStyle(color: Theme.of(context).colorScheme.error),
                    ),
                  ],
                  const SizedBox(height: 24),
                  SizedBox(
                    width: double.infinity,
                    child: FilledButton(
                      onPressed: loading.value ? null : submit,
                      child: loading.value
                          ? const SizedBox(
                              width: 18,
                              height: 18,
                              child: CircularProgressIndicator(strokeWidth: 2),
                            )
                          : const Text('Sign in'),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## Auth Callback Screen (Keycloak)

`flutter_appauth` resolves the redirect natively — no Flutter route is required. The
`/auth/callback` route is only used as a brief visual landing while the AppAuth library
finishes the exchange and the router rebuilds:

```dart
// lib/features/auth/presentation/auth_callback_screen.dart
import 'package:flutter/material.dart';

import '../../../shared/widgets/loading_indicator.dart';

class AuthCallbackScreen extends StatelessWidget {
  const AuthCallbackScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: LoadingIndicator(message: 'Signing you in...'),
    );
  }
}
```

---

## Secure Storage Notes

- `flutter_secure_storage` uses the Keychain on iOS (default accessibility: first unlock)
  and EncryptedSharedPreferences on Android (API 23+)
- Never store raw passwords — only tokens
- Hive boxes containing user-sensitive data should be encrypted using
  `HiveAesCipher` with a key generated once and stored in `flutter_secure_storage`
  (see `storage-patterns.md`)
- On logout, **all** Hive boxes and secure-storage keys must be cleared
