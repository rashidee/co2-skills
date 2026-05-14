# Specification Template — Flutter Mobile App with Riverpod

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint.
   Generated once per module from PRD.md.

```
spec/
├── SPECIFICATION.md
├── task/
│   └── SPEC.md
├── order/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values gathered
from context files.

Sections marked **[CONDITIONAL]** should only be included when the corresponding
integration is selected.

**CRITICAL: Sample code requirement.** Every widget/class in the spec MUST include a
complete, continuous Dart code sample in a fenced code block. The code must be
self-explanatory and directly usable as a reference for a coding agent. Do not describe
classes with bullet points alone — always accompany descriptions with full Dart code.

---

# Part A: Root SPECIFICATION.md

---

## Table of Contents

Generate a TOC with clickable Markdown anchor links to every H2 and H3 section.
Additionally, include a **Modules** section in the TOC that links to each module's
`SPEC.md` file:

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. Package Configuration](#2-package-configuration)
- ...all shared sections...

### Modules
- [Task](task/SPEC.md)
- [Order](order/SPEC.md)
- [Catalogue](catalogue/SPEC.md)
- ...(one link per module)...
```

Only include conditional sections that apply based on determined selections.

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}
**Project Slug**: {{PROJECT_SLUG}}
**App Display Name**: {{APP_TITLE}}
**Application ID**: {{APPLICATION_ID}}
**Framework**: Flutter Mobile (Android + iOS)
**Description**: {{APP_DESCRIPTION}}
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}

### Target Platforms
- **Android**: minSdk {{MIN_ANDROID_SDK}} (Android 6.0+), targetSdk 34
- **iOS**: deployment target {{MIN_IOS_VERSION}}+

### Optional Components (Auto-Determined)
**Backend API Base URL**: {{API_BASE_URL}}
**Authentication**: {{AUTH_TYPE}}
**WebSocket**: {{WEBSOCKET}}
**i18n**: {{I18N}}
**Analytics**: {{ANALYTICS}}
**Crashlytics**: {{CRASHLYTICS}}
**ImagePicker**: {{IMAGE_PICKER}}
**FilePicker**: {{FILE_PICKER}}
**Sharing**: {{SHARING}}
**Permissions**: {{PERMISSIONS}}

### Technology Stack
(Render the core stack version table from SKILL.md, plus selected optional integration versions)

### User Roles
(List each user role extracted from mockup role folders, with role constant and access scope.)

| Role | Description | Constant | Accessible Modules |
|------|-------------|----------|-------------------|
| Worker | Field worker | `WORKER` | Task, Job, Profile |
| Admin  | Full system | `ADMIN`  | All modules |

### Modules
(List each module from PRD.md, with type and link to its SPEC.md.)

| Module | Type | Stories | Versions | Spec |
|--------|------|---------|----------|------|
| Task | Business | 5 | 1.0.0, 1.0.4 | [SPEC](task/SPEC.md) |
| Profile | System | 3 | 1.0.0 | [SPEC](profile/SPEC.md) |

### Input Sources
- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md}}
- Mockup Screens: {{path to MOCKUP.html}}
```

---

## Section 2: Package Configuration

### 2.1 pubspec.yaml

Generate a complete `pubspec.yaml` inside a code block. Include all core and
conditional dependencies based on the determined selections.

```yaml
name: {{PROJECT_SLUG}}
description: {{APP_DESCRIPTION}}
publish_to: 'none'
version: {{APP_VERSION}}+1

environment:
  sdk: '>=3.5.0 <4.0.0'
  flutter: '>=3.24.0'

dependencies:
  flutter:
    sdk: flutter

  # State management
  flutter_riverpod: ^2.5.1
  hooks_riverpod: ^2.5.1
  flutter_hooks: ^0.20.5
  riverpod_annotation: ^2.3.5

  # Local storage
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  flutter_secure_storage: ^9.2.2
  path_provider: ^2.1.4

  # Networking
  dio: ^5.7.0
  dio_smart_retry: ^6.0.0
  pretty_dio_logger: ^1.4.0

  # Navigation
  go_router: ^14.2.7

  # Code generation models
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

  # Firebase + notifications
  firebase_core: ^3.6.0
  firebase_messaging: ^15.1.3
  flutter_local_notifications: ^17.2.3

  # Media / assets
  cached_network_image: ^3.4.1
  flutter_svg: ^2.0.10+1

  # UI extras
  pull_to_refresh: ^2.0.0
  font_awesome_flutter: ^10.7.0
  material_design_icons_flutter: ^7.0.7296

  # Utilities
  intl: ^0.19.0
  url_launcher: ^6.3.1
  flutter_dotenv: ^5.2.1
  package_info_plus: ^8.0.2

  # [If Auth = Keycloak]
  flutter_appauth: ^8.0.0+1
  openid_client: ^0.4.9

  # [If WebSocket = yes]
  web_socket_channel: ^3.0.1

  # [If Analytics = yes]
  firebase_analytics: ^11.3.3

  # [If Crashlytics = yes]
  firebase_crashlytics: ^4.1.3

  # [If ImagePicker = yes]
  image_picker: ^1.1.2
  permission_handler: ^11.3.1

  # [If FilePicker = yes]
  file_picker: ^8.1.2

  # [If Sharing = yes]
  share_plus: ^10.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
  build_runner: ^2.4.13
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.3
  hive_generator: ^2.0.1
  flutter_native_splash: ^2.4.1
  mocktail: ^1.0.4

flutter:
  uses-material-design: true
  assets:
    - .env.development
    - .env.production
    - assets/images/
    - assets/icons/

  fonts:
    - family: {{PRIMARY_FONT_FAMILY}}
      fonts:
        - asset: assets/fonts/{{PRIMARY_FONT_FAMILY}}-Regular.ttf
        - asset: assets/fonts/{{PRIMARY_FONT_FAMILY}}-Medium.ttf
          weight: 500
        - asset: assets/fonts/{{PRIMARY_FONT_FAMILY}}-Bold.ttf
          weight: 700
```

### 2.2 analysis_options.yaml

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "**/generated_plugin_registrant.dart"
    - "lib/generated/**"
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true

linter:
  rules:
    - prefer_const_constructors
    - prefer_const_literals_to_create_immutables
    - prefer_final_locals
    - prefer_final_in_for_each
    - avoid_print
    - use_super_parameters
    - require_trailing_commas
    - sort_pub_dependencies
```

### 2.3 build_runner Scripts

```bash
# One-shot code generation
dart run build_runner build --delete-conflicting-outputs

# Watch mode (rebuilds on save)
dart run build_runner watch --delete-conflicting-outputs

# Generate native splash screens (after editing pubspec config)
dart run flutter_native_splash:create
```

### 2.4 Environment Files

```env
# .env.development
API_BASE_URL=http://10.0.2.2:{{BACKEND_PORT}}/api/v1
API_TIMEOUT_MS=30000
API_RETRY_ATTEMPTS=3

# [If Auth = Keycloak]
KEYCLOAK_ISSUER=http://10.0.2.2:8180/realms/{{KEYCLOAK_REALM}}
KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
KEYCLOAK_REDIRECT_URI={{APPLICATION_ID}}:/oauth2redirect
KEYCLOAK_POST_LOGOUT_REDIRECT_URI={{APPLICATION_ID}}:/oauth2redirect

# [If WebSocket = yes]
WS_URL=ws://10.0.2.2:{{BACKEND_PORT}}/ws
```

```env
# .env.production
API_BASE_URL=https://api.example.com/api/v1
API_TIMEOUT_MS=30000
API_RETRY_ATTEMPTS=3

# [If Auth = Keycloak]
KEYCLOAK_ISSUER=https://sso.example.com/realms/{{KEYCLOAK_REALM}}
KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
KEYCLOAK_REDIRECT_URI={{APPLICATION_ID}}:/oauth2redirect
KEYCLOAK_POST_LOGOUT_REDIRECT_URI={{APPLICATION_ID}}:/oauth2redirect
```

---

## Section 3: Application Configuration

### 3.1 .gitignore (Flutter)

```gitignore
# Miscellaneous
*.class
*.log
*.pyc
*.swp
.DS_Store
.atom/
.buildlog/
.history
.svn/
migrate_working_dir/

# IntelliJ / VSCode
.idea/
.vscode/
*.iml
*.ipr
*.iws

# Flutter / Dart / Pub-related
**/doc/api/
**/ios/Flutter/.last_build_id
.dart_tool/
.flutter-plugins
.flutter-plugins-dependencies
.packages
.pub-cache/
.pub/
/build/
/coverage/
pubspec.lock

# Symbolication
app.*.symbols
app.*.map.json

# Obfuscation
app.*.symbols

# Android
**/android/**/gradle-wrapper.jar
**/android/.gradle
**/android/captures/
**/android/gradlew
**/android/gradlew.bat
**/android/local.properties
**/android/**/GeneratedPluginRegistrant.java
**/android/key.properties
**/android/app/release/
**/android/app/google-services.json

# iOS
**/ios/**/*.mode1v3
**/ios/**/*.mode2v3
**/ios/**/*.moved-aside
**/ios/**/*.pbxuser
**/ios/**/*.perspectivev3
**/ios/**/Pods/
**/ios/**/.symlinks/
**/ios/**/profile
**/ios/**/xcuserdata
**/ios/.generated/
**/ios/Flutter/App.framework
**/ios/Flutter/Flutter.framework
**/ios/Flutter/Flutter.podspec
**/ios/Flutter/Generated.xcconfig
**/ios/Flutter/ephemeral
**/ios/Flutter/app.flx
**/ios/Flutter/app.zip
**/ios/Flutter/flutter_assets/
**/ios/Flutter/flutter_export_environment.sh
**/ios/ServiceDefinitions.json
**/ios/Runner/GeneratedPluginRegistrant.*
**/ios/Runner/GoogleService-Info.plist

# Environment (never commit secrets)
.env
.env.local
.env.development
.env.production
.env.*.local
```

### 3.2 main.dart

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';

import 'firebase_options.dart';
import 'app.dart';
import 'core/local_storage/hive_setup.dart';
import 'core/notification/notification_service.dart';

@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Load environment-specific .env
  final env = kReleaseMode ? '.env.production' : '.env.development';
  await dotenv.load(fileName: env);

  // Initialize Firebase
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

  // Initialize Hive + register adapters
  await HiveSetup.init();

  // Initialize local notifications
  await NotificationService.instance.init();

  runApp(const ProviderScope(child: MyApp()));
}
```

### 3.3 Android Configuration Notes

```text
android/app/build.gradle:
- applicationId "{{APPLICATION_ID}}"
- minSdkVersion {{MIN_ANDROID_SDK}}
- targetSdkVersion 34
- compileSdkVersion 34

android/app/src/main/AndroidManifest.xml:
- <uses-permission android:name="android.permission.INTERNET" />
- <uses-permission android:name="android.permission.POST_NOTIFICATIONS" /> <!-- Android 13+ -->
- [If ImagePicker = yes] <uses-permission android:name="android.permission.CAMERA" />
- Deep link intent filter for Keycloak callback:
    <intent-filter android:autoVerify="true">
      <action android:name="android.intent.action.VIEW"/>
      <category android:name="android.intent.category.DEFAULT"/>
      <category android:name="android.intent.category.BROWSABLE"/>
      <data android:scheme="{{APPLICATION_ID}}"/>
    </intent-filter>
- Default notification channel meta-data (FCM):
    <meta-data
      android:name="com.google.firebase.messaging.default_notification_channel_id"
      android:value="default" />

android/app/google-services.json:
- Add file (gitignored — pulled from Firebase project)
- Add `apply plugin: 'com.google.gms.google-services'` at the bottom of build.gradle
- Add classpath in android/build.gradle: 'com.google.gms:google-services:4.4.2'
```

### 3.4 iOS Configuration Notes

```text
ios/Runner/Info.plist:
- CFBundleURLTypes for Keycloak callback (URL scheme = {{APPLICATION_ID}})
- [If ImagePicker = yes] NSCameraUsageDescription, NSPhotoLibraryUsageDescription
- [If Permissions = yes] NSLocationWhenInUseUsageDescription (if location used)
- UIBackgroundModes: remote-notification (for FCM background delivery)

ios/Runner/AppDelegate.swift:
- Register for remote notifications and pass FCM token to Flutter
- Set `application.registerForRemoteNotifications()` and the
  `UNUserNotificationCenter.current().delegate = self`

ios/Runner/GoogleService-Info.plist:
- Add file (gitignored — pulled from Firebase project)
- Drag into Runner target in Xcode

ios/Podfile:
- platform :ios, '{{MIN_IOS_VERSION}}'
- post_install hook adds GCC_PREPROCESSOR_DEFINITIONS for flutter_local_notifications
```

### 3.5 flutter_native_splash Configuration (pubspec.yaml)

```yaml
flutter_native_splash:
  color: "{{PAGE_BACKGROUND_HEX}}"
  image: assets/images/splash.png
  color_dark: "#121212"
  image_dark: assets/images/splash_dark.png
  android_12:
    image: assets/images/splash_android12.png
    icon_background_color: "{{PRIMARY_COLOR_HEX}}"
  ios: true
  android: true
  web: false
```

Run: `dart run flutter_native_splash:create`

---

## Section 4: Directory Structure

Generate the full `lib/` directory tree. Use actual module names from PRD.md and MODEL.md.
Follow the feature-based structure from `references/routing-patterns.md`.

```
lib/
├── main.dart
├── app.dart                              # MaterialApp.router + theme + locale
├── firebase_options.dart                 # Generated by `flutterfire configure`
├── router/
│   └── app_router.dart                   # go_router configuration
├── core/
│   ├── api/
│   │   ├── dio_provider.dart             # Dio instance + interceptors
│   │   ├── api_failure.dart              # Sealed-class error type
│   │   └── auth_interceptor.dart         # [If Auth != none] Bearer token injector
│   ├── auth/
│   │   ├── auth_repository.dart          # signIn/signOut/refresh
│   │   ├── auth_provider.dart            # Riverpod auth state
│   │   ├── auth_state.dart               # Sealed-class auth state (freezed)
│   │   └── roles.dart                    # Role constants
│   ├── local_storage/
│   │   ├── hive_setup.dart               # Hive.initFlutter + adapter registration
│   │   ├── secure_storage_provider.dart  # flutter_secure_storage wrapper
│   │   └── box_provider.dart             # Typed Box<T> providers
│   ├── notification/
│   │   ├── notification_service.dart     # FCM + local notifications init
│   │   ├── notification_handler.dart     # Foreground / background / tap handlers
│   │   └── notification_navigation.dart  # Routes deep-linked from notifications
│   ├── theme/
│   │   ├── app_theme.dart                # lightTheme + darkTheme
│   │   ├── app_colors.dart               # Color tokens from mockup
│   │   └── app_typography.dart           # TextTheme tokens
│   ├── env/
│   │   └── env_config.dart               # Typed accessor over dotenv
│   └── utils/
│       ├── validators.dart               # Form field validators
│       ├── formatters.dart               # intl-based formatters
│       └── url_helper.dart               # url_launcher wrappers
├── shared/
│   ├── widgets/
│   │   ├── app_scaffold.dart
│   │   ├── loading_indicator.dart
│   │   ├── empty_state.dart
│   │   ├── error_state.dart
│   │   ├── confirm_dialog.dart
│   │   ├── status_chip.dart
│   │   ├── search_bar_field.dart
│   │   ├── pull_to_refresh_list.dart
│   │   ├── infinite_scroll_list.dart
│   │   ├── app_network_image.dart       # Wraps cached_network_image
│   │   ├── cached_svg_icon.dart         # Wraps flutter_svg
│   │   └── shell_screen.dart            # AppShell with bottom navigation
│   ├── form/
│   │   ├── form_text_field.dart
│   │   ├── form_dropdown_field.dart
│   │   ├── form_date_picker_field.dart
│   │   ├── form_switch_field.dart
│   │   └── form_image_picker_field.dart  # [If ImagePicker = yes]
│   └── navigation/
│       └── nav_config.dart                # Bottom-nav / drawer items
├── features/
│   ├── auth/                              # [If Auth != none]
│   │   ├── presentation/
│   │   │   ├── login_screen.dart          # [If Auth = Local]
│   │   │   └── auth_callback_screen.dart  # [If Auth = Keycloak]
│   │   └── auth.dart                      # Barrel
│   ├── dashboard/
│   │   ├── presentation/
│   │   │   └── dashboard_screen.dart
│   │   └── dashboard.dart
│   ├── {{module1}}/
│   │   ├── data/
│   │   │   ├── {{module1}}_repository.dart
│   │   │   └── {{module1}}_remote_data_source.dart
│   │   ├── domain/
│   │   │   └── {{module1}}.dart            # @freezed model + Hive adapter
│   │   ├── application/
│   │   │   └── {{module1}}_providers.dart  # Riverpod providers (generated)
│   │   ├── presentation/
│   │   │   ├── {{module1}}_list_screen.dart
│   │   │   ├── {{module1}}_detail_screen.dart
│   │   │   └── {{module1}}_form_screen.dart
│   │   └── {{module1}}.dart                # Barrel
│   └── {{module2}}/
│       └── (same structure)
└── l10n/                                   # [If i18n = yes]
    ├── app_en.arb
    ├── app_ms.arb
    └── intl_helper.dart
```

---

## Section 5: Theme Configuration

Generate the complete Material 3 `ThemeData` using design tokens extracted from MOCKUP.html.
Reference `references/component-patterns.md` for the theme setup pattern.

Include:
- `lib/core/theme/app_colors.dart` — color tokens with actual hex values from mockup
- `lib/core/theme/app_typography.dart` — TextTheme tokens from mockup
- `lib/core/theme/app_theme.dart` — `ThemeData lightTheme` + `ThemeData darkTheme` factory,
  component themes (ElevatedButtonTheme, AppBarTheme, CardTheme, InputDecorationTheme)
- Design token mapping table showing mockup CSS property → ThemeData property

**app.dart integration:**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'core/theme/app_theme.dart';
import 'router/app_router.dart';
import 'shared/providers/theme_mode_provider.dart';

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeMode = ref.watch(themeModeProvider);
    final router = ref.watch(appRouterProvider);

    return MaterialApp.router(
      title: '{{APP_TITLE}}',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.light(),
      darkTheme: AppTheme.dark(),
      themeMode: themeMode,
      routerConfig: router,
      // [If i18n = yes]
      // localizationsDelegates: AppLocalizations.localizationsDelegates,
      // supportedLocales: AppLocalizations.supportedLocales,
    );
  }
}
```

---

## Section 6: Authentication Configuration [CONDITIONAL — Include only if Auth != none]

**[If Auth = Keycloak]:**
Refer to `references/security-patterns.md` for the complete content.
Must cover:
- `lib/core/auth/auth_repository.dart` — wraps `FlutterAppAuth` for sign-in/refresh
- `lib/core/auth/auth_state.dart` — sealed-class auth state via freezed
- `lib/core/auth/auth_provider.dart` — `AuthNotifier extends AsyncNotifier<AuthState>`
- `lib/core/auth/roles.dart` — role constants
- `lib/core/api/auth_interceptor.dart` — Dio interceptor that injects Bearer token
- Token persistence in `flutter_secure_storage`
- AppAuth URL scheme registration (Android `AndroidManifest.xml` + iOS `Info.plist`)
- Auth callback handled natively by `flutter_appauth` (no Flutter route needed)
- Silent token refresh logic on app startup

**[If Auth = Local]:**
Refer to `references/security-patterns.md` (Local JWT section) for the complete content.
Must cover:
- `lib/features/auth/presentation/login_screen.dart` — email/password form
- `lib/core/auth/auth_repository.dart` — POST `/auth/login`, `/auth/refresh`
- `lib/core/auth/auth_provider.dart` — Riverpod auth state with login/logout actions
- `lib/core/api/auth_interceptor.dart` — reads token from `flutter_secure_storage`,
  refreshes on 401
- go_router `redirect` guard that pushes `/login` when unauthenticated

---

## Section 7: Router Configuration

Refer to `references/routing-patterns.md` for the complete route tree.
Must cover:
- `lib/router/app_router.dart` — complete `GoRouter(...)` with ALL routes from PRD.md
- `ShellRoute` for the bottom-navigation scaffold
- `redirect` callback for auth + role enforcement
- Deep link routes (e.g., `/notifications/:id`) for FCM payload navigation
- The exact module-based URL paths (e.g., `/task`, NOT `/worker/task`)
- 404 and unauthorized routes

---

## Section 8: API Client Configuration

```dart
// lib/core/api/dio_provider.dart
import 'package:dio/dio.dart';
import 'package:dio_smart_retry/dio_smart_retry.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import 'auth_interceptor.dart';

part 'dio_provider.g.dart';

@Riverpod(keepAlive: true)
Dio dio(DioRef ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: dotenv.env['API_BASE_URL']!,
      connectTimeout: Duration(
        milliseconds: int.parse(dotenv.env['API_TIMEOUT_MS'] ?? '30000'),
      ),
      receiveTimeout: Duration(
        milliseconds: int.parse(dotenv.env['API_TIMEOUT_MS'] ?? '30000'),
      ),
      headers: {'Content-Type': 'application/json'},
    ),
  );

  // Auth interceptor — see Section 6
  dio.interceptors.add(AuthInterceptor(ref));

  // Smart retry — exponential backoff on 5xx and network errors,
  // only for idempotent methods (GET, PUT, DELETE)
  dio.interceptors.add(
    RetryInterceptor(
      dio: dio,
      retries: int.parse(dotenv.env['API_RETRY_ATTEMPTS'] ?? '3'),
      retryDelays: const [
        Duration(seconds: 1),
        Duration(seconds: 2),
        Duration(seconds: 4),
      ],
      retryableExtraStatuses: const {502, 503, 504},
    ),
  );

  // Verbose logging — dev only
  if (kDebugMode) {
    dio.interceptors.add(
      PrettyDioLogger(
        requestHeader: true,
        requestBody: true,
        responseBody: true,
        responseHeader: false,
        compact: false,
      ),
    );
  }

  return dio;
}
```

```dart
// lib/core/api/api_failure.dart
import 'package:dio/dio.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_failure.freezed.dart';

@freezed
sealed class ApiFailure with _$ApiFailure {
  const factory ApiFailure.network() = _Network;
  const factory ApiFailure.unauthorized() = _Unauthorized;
  const factory ApiFailure.forbidden() = _Forbidden;
  const factory ApiFailure.notFound() = _NotFound;
  const factory ApiFailure.server({int? status, String? message}) = _Server;
  const factory ApiFailure.unknown(Object error) = _Unknown;

  factory ApiFailure.fromDioException(DioException error) {
    if (error.type == DioExceptionType.connectionTimeout ||
        error.type == DioExceptionType.receiveTimeout ||
        error.type == DioExceptionType.sendTimeout ||
        error.type == DioExceptionType.connectionError) {
      return const ApiFailure.network();
    }
    final status = error.response?.statusCode ?? 0;
    if (status == 401) return const ApiFailure.unauthorized();
    if (status == 403) return const ApiFailure.forbidden();
    if (status == 404) return const ApiFailure.notFound();
    if (status >= 500) {
      final body = error.response?.data;
      final message = body is Map<String, dynamic> ? body['message'] as String? : null;
      return ApiFailure.server(status: status, message: message);
    }
    return ApiFailure.unknown(error);
  }
}
```

---

## Section 9: Local Storage (Hive)

Refer to `references/storage-patterns.md`. Cover:
- `lib/core/local_storage/hive_setup.dart` — `Hive.initFlutter()`, register adapters,
  open boxes (`settings`, `user_profile`, `<module>_cache`)
- `lib/core/local_storage/secure_storage_provider.dart` — `FlutterSecureStorage`
  wrapper exposed as Riverpod provider
- `lib/core/local_storage/box_provider.dart` — typed `Box<T>` providers
- Encryption pattern using `HiveAesCipher` with key from secure storage
- Migration strategy on schema changes (clear-and-refetch vs. lazy migration)

---

## Section 10: Riverpod Setup

Refer to `references/state-patterns.md`. Cover:
- `ProviderScope` wrapping `MyApp` in `main.dart`
- `riverpod_generator` configuration (`build.yaml` snippet)
- Provider naming conventions (`<Entity>RepositoryProvider`, `<Entity>ListProvider`,
  `<Entity>ByIdProvider(id)`, `<Entity>FormControllerProvider`)
- `AsyncNotifier` / `AsyncNotifierProvider.family` patterns
- Side-effect mutation pattern (`Notifier` exposing `create()`, `update()`, `delete()`)
- Cache invalidation via `ref.invalidate(...)` and `ref.refresh(...)`
- Listening pattern in widgets via `ref.watch(...)` + `AsyncValue.when(...)`

---

## Section 11: Shared Layouts

For each shell, provide:
1. The constructor signature
2. The complete widget

- **AppShellScreen**: `Scaffold` with `body` from `ShellRoute` outlet and `BottomNavigationBar`
  driven by `nav_config.dart` (role-filtered)
- **AuthShell**: Centered `Scaffold` for login / forgot-password / Keycloak callback
- **PublicShell**: Minimal scaffold for unauthenticated content (if any)

---

## Section 12: Shared Widgets

For each shared widget, provide the constructor signature and complete Dart code.
Reference `references/component-patterns.md` for implementations.

| Widget | Purpose |
|---|---|
| `AppScaffold` | Standard AppBar + body + optional FAB |
| `LoadingIndicator` | Full-screen CircularProgressIndicator |
| `EmptyState` | Empty list placeholder with icon and message |
| `ErrorState` | Full-screen error with retry button |
| `ConfirmDialog` | `showDialog`-based confirmation |
| `StatusChip` | Colored Chip for enum status values |
| `SearchBarField` | Debounced TextField for searches |
| `PullToRefreshList` | `SmartRefresher` + `ListView` wrapper |
| `InfiniteScrollList` | ScrollController-driven pagination wrapper |
| `AppNetworkImage` | `CachedNetworkImage` with placeholder + error fallback |
| `CachedSvgIcon` | `SvgPicture.asset` / `.network` with cache |

---

## Section 13: Navigation Configuration

Generate `lib/shared/navigation/nav_config.dart` with the actual navigation items
from the mockup nav files. For each role:
- List all nav items that role can see
- Include the icon (mix of `Icons.*`, `FontAwesomeIcons.*`, `MdiIcons.*` as appropriate)
- Bottom-nav routes use module-based paths only (e.g., `/task`, `/profile`)

```dart
// lib/shared/navigation/nav_config.dart
import 'package:flutter/material.dart';
import 'package:font_awesome_flutter/font_awesome_flutter.dart';
import 'package:material_design_icons_flutter/material_design_icons_flutter.dart';

import '../../core/auth/roles.dart';

class NavItem {
  const NavItem({
    required this.label,
    required this.path,
    required this.icon,
    required this.activeIcon,
    this.requiredRoles,
  });

  final String label;
  final String path;
  final IconData icon;
  final IconData activeIcon;
  /// If null, accessible to all authenticated users
  final List<String>? requiredRoles;
}

const navigationItems = <NavItem>[
  NavItem(
    label: 'Dashboard',
    path: '/dashboard',
    icon: Icons.dashboard_outlined,
    activeIcon: Icons.dashboard,
  ),
  NavItem(
    label: 'Tasks',
    path: '/task',
    icon: MdiIcons.formatListChecks,
    activeIcon: MdiIcons.formatListChecks,
    requiredRoles: [Roles.worker, Roles.admin],
  ),
  NavItem(
    label: 'Profile',
    path: '/profile',
    icon: FontAwesomeIcons.user,
    activeIcon: FontAwesomeIcons.userLarge,
  ),
];
```

---

## Section 14: Form Infrastructure

Reference `references/component-patterns.md` for the complete reusable form-field widgets.
Describe:
- `Form` + `GlobalKey<FormState>` pattern
- All `Form*Field` widgets with full constructor signatures
- Reset pattern for edit modes (re-initialize controllers in `initState` / `didUpdateWidget`)
- Loading state on submit button (replace text with `CircularProgressIndicator`)

```dart
// lib/core/utils/validators.dart
class Validators {
  Validators._();

  static String? required(String? value, {String label = 'This field'}) {
    if (value == null || value.trim().isEmpty) return '$label is required';
    return null;
  }

  static String? minLength(String? value, int min, {String label = 'This field'}) {
    if (value == null) return null;
    if (value.length < min) return '$label must be at least $min characters';
    return null;
  }

  static String? maxLength(String? value, int max, {String label = 'This field'}) {
    if (value == null) return null;
    if (value.length > max) return '$label must be at most $max characters';
    return null;
  }

  static String? email(String? value) {
    if (value == null || value.isEmpty) return null;
    final pattern = RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$');
    if (!pattern.hasMatch(value)) return 'Enter a valid email address';
    return null;
  }

  static String? url(String? value) {
    if (value == null || value.isEmpty) return null;
    final parsed = Uri.tryParse(value);
    if (parsed == null || !(parsed.isScheme('http') || parsed.isScheme('https'))) {
      return 'Enter a valid URL';
    }
    return null;
  }

  static FormFieldValidator<String> compose(List<FormFieldValidator<String>> validators) {
    return (value) {
      for (final v in validators) {
        final result = v(value);
        if (result != null) return result;
      }
      return null;
    };
  }
}
```

---

## Section 15: Error Handling Strategy

Describe:
- `FlutterError.onError` + `PlatformDispatcher.instance.onError` in `main.dart`
- Optional Crashlytics forwarding: `FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterError;`
- Dio response interceptor mapping `DioException` → `ApiFailure`
- Riverpod `AsyncValue.when(loading: ..., error: ..., data: ...)` pattern in screens
- SnackBar via `ScaffoldMessenger` for transient feedback

```dart
// lib/shared/widgets/error_state.dart
import 'package:flutter/material.dart';

import '../../core/api/api_failure.dart';

class ErrorState extends StatelessWidget {
  const ErrorState({super.key, required this.error, this.onRetry});

  final Object error;
  final VoidCallback? onRetry;

  String _messageFor(Object error) {
    if (error is ApiFailure) {
      return switch (error) {
        ApiFailure(:final _) when error is _Network => 'No internet connection. Please try again.',
        ApiFailure() when error is _Unauthorized => 'Your session has expired. Please sign in again.',
        ApiFailure() when error is _Forbidden => 'You do not have permission to perform this action.',
        ApiFailure() when error is _NotFound => 'The requested resource was not found.',
        ApiFailure() when error is _Server => 'A server error occurred. Please try again later.',
        _ => 'An unexpected error occurred.',
      };
    }
    return error.toString();
  }

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.error_outline,
              size: 56,
              color: Theme.of(context).colorScheme.error,
            ),
            const SizedBox(height: 12),
            Text(
              _messageFor(error),
              style: Theme.of(context).textTheme.bodyLarge,
              textAlign: TextAlign.center,
            ),
            if (onRetry != null) ...[
              const SizedBox(height: 16),
              FilledButton.icon(
                onPressed: onRetry,
                icon: const Icon(Icons.refresh),
                label: const Text('Retry'),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## Section 16: Notification System

Refer to `references/notification-patterns.md` for the full FCM + flutter_local_notifications
architecture. Cover:
- `NotificationService.init()` lifecycle
- Permission request flow (iOS + Android 13+)
- Foreground / background / terminated handlers
- Local notification display for foreground FCM messages
- Deep link from notification tap → `router.go('/...')`
- FCM token registration with backend on first launch + refresh
- Topic subscription (if PRD describes topic-based broadcasts)
- Scheduled local notifications (`zonedSchedule`) for in-app reminders

---

## Section 17: Theming & Dark Mode

```dart
// lib/shared/providers/theme_mode_provider.dart
import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'theme_mode_provider.g.dart';

const _settingsBox = 'settings';
const _themeModeKey = 'themeMode';

@Riverpod(keepAlive: true)
class ThemeModeController extends _$ThemeModeController {
  late Box _box;

  @override
  ThemeMode build() {
    _box = Hive.box(_settingsBox);
    final stored = _box.get(_themeModeKey) as String?;
    return switch (stored) {
      'dark' => ThemeMode.dark,
      'light' => ThemeMode.light,
      _ => ThemeMode.system,
    };
  }

  Future<void> toggle() async {
    final next = switch (state) {
      ThemeMode.light => ThemeMode.dark,
      ThemeMode.dark => ThemeMode.light,
      ThemeMode.system => ThemeMode.dark,
    };
    await _box.put(_themeModeKey, next.name);
    state = next;
  }
}
```

---

## Section 18: Testing Strategy

Overview only — detailed per-feature test patterns in each module's SPEC.md.

- **Unit tests**: Pure Dart logic (validators, formatters, repository mappers)
- **Provider tests**: Use `ProviderContainer` with overrides (mock repositories)
- **Widget tests**: `flutter_test` + `ProviderScope` overrides
- **Integration tests**: `integration_test` package for end-to-end flows
- **Mocking**: `mocktail` (preferred — no codegen) for repository / Dio mocks

```yaml
# pubspec.yaml — add under dev_dependencies if not present
integration_test:
  sdk: flutter
```

---

## Section 19: Build & Distribution

### Android Release Build

```bash
# Generate keystore (one-time)
keytool -genkey -v -keystore upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload

# Configure ~/.gradle/gradle.properties:
# RELEASE_STORE_FILE=/path/to/upload-keystore.jks
# RELEASE_STORE_PASSWORD=...
# RELEASE_KEY_ALIAS=upload
# RELEASE_KEY_PASSWORD=...

flutter build appbundle --release          # Google Play
flutter build apk --release --split-per-abi # Sideload
```

### iOS Release Build

```bash
flutter build ipa --release
# Or open ios/Runner.xcworkspace in Xcode and Archive
```

### Native Splash & Icons

```bash
dart run flutter_native_splash:create
# Optional launcher icons (add flutter_launcher_icons dev dependency)
dart run flutter_launcher_icons
```

### ProGuard Rules (Android Release)

```text
# android/app/proguard-rules.pro

# Firebase
-keep class com.google.firebase.** { *; }
-keep class com.google.android.gms.** { *; }

# AppAuth (Keycloak)
-keep class net.openid.appauth.** { *; }

# flutter_local_notifications
-keep class com.dexterous.** { *; }

# Keep Dio generic types
-keepattributes Signature
```

---

## Section 20: Internationalisation [CONDITIONAL — Include only if i18n = yes]

```yaml
# pubspec.yaml
flutter:
  generate: true
```

```yaml
# l10n.yaml (project root)
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
synthetic-package: false
output-dir: lib/generated
```

```json
// lib/l10n/app_en.arb
{
  "@@locale": "en",
  "appTitle": "{{APP_TITLE}}",
  "actionRetry": "Retry",
  "errorGeneric": "An unexpected error occurred."
}
```

Wire `MaterialApp.router` localization delegates (already shown in Section 5) and add
a locale-switch provider persisted in Hive `settings` box.

---

## Section 21: WebSocket Integration [CONDITIONAL — Include only if WebSocket = yes]

```dart
// lib/core/realtime/socket_provider.dart
import 'dart:async';

import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:web_socket_channel/web_socket_channel.dart';

part 'socket_provider.g.dart';

@Riverpod(keepAlive: true)
Stream<dynamic> socketEvents(SocketEventsRef ref) async* {
  final url = dotenv.env['WS_URL']!;
  while (true) {
    final channel = WebSocketChannel.connect(Uri.parse(url));
    try {
      ref.onDispose(channel.sink.close);
      await for (final event in channel.stream) {
        yield event;
      }
    } catch (_) {
      // Backoff before reconnect
      await Future<void>.delayed(const Duration(seconds: 3));
    }
  }
}
```

---

## Section 22: Firebase Initialization

```bash
# One-time CLI flow during scaffolding
dart pub global activate flutterfire_cli
flutterfire configure --project={{FIREBASE_PROJECT_ID}}
```

This produces `lib/firebase_options.dart` and downloads `google-services.json` /
`GoogleService-Info.plist` automatically. Both native files must be gitignored.

```dart
// main.dart excerpt (already shown above)
await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
```

---

# Part B: Per-Module SPEC.md Template

Each module from PRD.md and MODEL.md gets its own `<module-name>/SPEC.md` file.

---

## Module SPEC.md Structure

```markdown
# {{ModuleName}} — Module Specification

> Part of [{{APPLICATION_NAME}} Technical Specification](../SPECIFICATION.md)

## Overview

**Module:** {{ModuleName}}
**Feature Path:** `lib/features/{{moduleName}}/`
**Type:** {{System Module | Business Module}}
**Backend Endpoint Base:** `/{{api-path}}` (inferred from model/API context)
**Cache Box:** `{{moduleName}}_cache` (if module is cached offline)
```

---

### Traceability

```markdown
## Traceability

### User Stories
| ID | Version | Description |
|---|---|---|
| USA000030 | v1.0.0 | Create/update task |
| USA000033 | v1.0.0 | View list of tasks |

### Non-Functional Requirements
| ID | Version | Description |
|---|---|---|
| NFRA00021 | v1.0.0 | Task list paginated with 20 items per page |
| NFRA00024 | v1.0.0 | Task title max 100 characters |

### Constraints
| ID | Version | Description |
|---|---|---|
| CONSA0012 | v1.0.0 | Status values: DRAFT, IN_PROGRESS, COMPLETED |

### Removed / Replaced
| ID | Removed In | Replaced By | Reason |
|---|---|---|---|
| CONSA0012 (READY status) | v1.0.4 | — | Auto-computed status; READY removed |
_or "None." if no removals_

### Data Sources
| Artifact | Reference |
|---|---|
| DB Table | `task` |
| Mockup Screen | `worker/task/task-list.html` |
```

---

### Freezed Model

```markdown
## Freezed Model

File: `lib/features/{{moduleName}}/domain/{{moduleName}}.dart`

(Complete @freezed data class — field-for-field from MODEL.md, with fromJson/toJson and
optional Hive adapter annotations if module is cached offline)
```

All enum values must match PRD.md constraints exactly. Include the generic
`PaginatedResponse<T>` type in `lib/core/api/` (shared), not per-module.

```dart
// Example
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:hive/hive.dart';

part 'task.freezed.dart';
part 'task.g.dart';

enum TaskStatus {
  @JsonValue('DRAFT') draft,
  @JsonValue('IN_PROGRESS') inProgress,
  @JsonValue('COMPLETED') completed,
}

@freezed
@HiveType(typeId: 1)
class Task with _$Task {
  @HiveField(0) const factory Task({
    @HiveField(0) required String id,
    @HiveField(1) required String title,
    @HiveField(2) String? description,
    @HiveField(3) required TaskStatus status,
    @HiveField(4) required DateTime dueDate,
    @HiveField(5) required DateTime createdAt,
    @HiveField(6) required DateTime updatedAt,
  }) = _Task;

  factory Task.fromJson(Map<String, dynamic> json) => _$TaskFromJson(json);
}
```

---

### Validation

```markdown
## Validation

File: `lib/features/{{moduleName}}/presentation/{{moduleName}}_form_validators.dart`

(Validator closures derived from PRD.md NFRs and Constraints)
```

Rules must be explicitly derived from tagged NFR IDs:
- Character limits → `Validators.maxLength`
- URL format → `Validators.url`
- Required fields → `Validators.required`
- Cross-field rules → composed in the `onSubmit` callback after `formKey.currentState!.validate()`

---

### Repository

```markdown
## Repository

File: `lib/features/{{moduleName}}/data/{{moduleName}}_repository.dart`

(Complete repository class with all methods matching user stories)
```

- `Future<PaginatedResponse<{{Entity}}>> list({{ListParams}} params)` — list/search user stories
- `Future<{{Entity}}> getById(String id)` — view detail user stories
- `Future<{{Entity}}> create({{EntityCreate}} payload, {File? imageFile})` — create
- `Future<{{Entity}}> update(String id, {{EntityUpdate}} payload, {File? imageFile})` — update
- `Future<void> delete(String id)` — delete
- `String imageUrl(String id)` — if the module has image fields

---

### Riverpod Providers

```markdown
## Riverpod Providers

File: `lib/features/{{moduleName}}/application/{{moduleName}}_providers.dart`

(Complete generated providers using @riverpod annotation)
```

- `<entity>RepositoryProvider` — exposes the repository instance
- `<entity>ListProvider(<params>)` — `AsyncNotifierProvider.family` for paginated list
- `<entity>ByIdProvider(String id)` — `FutureProvider.family` for single item
- `<entity>FormControllerProvider` — `Notifier` exposing `create`/`update`/`delete` with
  cache invalidation via `ref.invalidate(<entity>ListProvider)`

---

### Screens

```markdown
## Screens

Files:
- `lib/features/{{moduleName}}/presentation/{{moduleName}}_list_screen.dart`
- `lib/features/{{moduleName}}/presentation/{{moduleName}}_detail_screen.dart`
- `lib/features/{{moduleName}}/presentation/{{moduleName}}_form_screen.dart`

(Complete screen widget implementations matching the mockup screens)
```

**CRITICAL — screen widget requirements:**
1. Extend `ConsumerWidget` or `HookConsumerWidget`
2. Use `ref.watch(<entity>ListProvider(params))` and unwrap with `AsyncValue.when(...)`
3. Use `context.go(...)` / `context.push(...)` for navigation (go_router)
4. List screen: wrap with `SmartRefresher` (pull_to_refresh) and call `ref.invalidate(...)`
   in `onRefresh`
5. Form screen: distinguish create vs edit via the `:id` path parameter
6. Use `ConfirmDialog.show(context, ...)` for destructive actions
7. Use `ScaffoldMessenger` for transient success/error toasts

Form screen pattern for create/edit:
```dart
class TaskFormScreen extends HookConsumerWidget {
  const TaskFormScreen({super.key, this.taskId});
  final String? taskId;
  bool get isEditMode => taskId != null;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // If isEditMode, watch taskByIdProvider(taskId!) and prefill form
    // Use formKey + TextEditingControllers for fields
    // ...
  }
}
```

---

### Form Widget

```markdown
## Form Widget

File: `lib/features/{{moduleName}}/presentation/widgets/{{moduleName}}_form.dart`

(Complete form widget using Form + GlobalKey<FormState> + reusable field widgets)
```

---

### Route Definitions

```markdown
## Route Definitions

Add to `lib/router/app_router.dart`:

(Complete GoRoute entries for this module — with redirect guards matching the mockup
role folder access control)
```

Example:
```dart
GoRoute(
  path: '/task',
  name: 'task-list',
  redirect: (context, state) => _requireRoles(ref, [Roles.worker, Roles.admin]),
  builder: (context, state) => const TaskListScreen(),
  routes: [
    GoRoute(
      path: 'create',
      name: 'task-create',
      builder: (context, state) => const TaskFormScreen(),
    ),
    GoRoute(
      path: ':id',
      name: 'task-detail',
      builder: (context, state) => TaskDetailScreen(taskId: state.pathParameters['id']!),
      routes: [
        GoRoute(
          path: 'edit',
          name: 'task-edit',
          builder: (context, state) => TaskFormScreen(taskId: state.pathParameters['id']),
        ),
      ],
    ),
  ],
),
```

---

### Navigation Items

```markdown
## Navigation Items

Add to `lib/shared/navigation/nav_config.dart`:

(NavItem entries for this module, per role)
```

---

### Status Chip Map (if module has status enum)

```markdown
## Status Configuration

```dart
// Status display map for use with <StatusChip />
const taskStatusDisplay = {
  TaskStatus.draft:      (label: 'Draft',       color: Colors.grey),
  TaskStatus.inProgress: (label: 'In Progress', color: Colors.blue),
  TaskStatus.completed:  (label: 'Completed',   color: Colors.green),
};
```

---

### CRITICAL Rules for Module SPEC.md

1. **Use real field names** from MODEL.md — not `fieldOne`/`fieldTwo`
2. **Use real table/collection names** from MODEL.md
3. **Map every user story** to a specific repository method or provider
4. **Map every NFR constraint** to a validator rule or widget behaviour
5. **Use module-based URLs** — `/task`, NOT `/worker/task`
6. **Role guards** come from mockup role folder structure → `redirect` callback on routes
7. **Version-tag every ID** in traceability tables
8. **Use null-safe Dart** — no `dynamic` except at the JSON boundary
9. **All code samples must be complete** — no `// ...` gaps
10. **Form screen handles both create and edit** via the `:id` path parameter unless
    the mockup has clearly separate screens
