# Storage Patterns — Hive Local Database + Dio HTTP Client

This reference describes how Hive (local NoSQL storage) and Dio (HTTP client) are
configured and used together in the spec. Include this content in Sections 8 and 9 of
the generated specification.

---

## Hive Setup

### Initialization

```dart
// lib/core/local_storage/hive_setup.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';

import '../../features/task/domain/task.dart';
import '../../features/profile/domain/profile.dart';

class HiveSetup {
  HiveSetup._();

  static const _settingsBox = 'settings';
  static const _userProfileBox = 'user_profile';
  static const _taskCacheBox = 'task_cache';
  static const _encryptionKeyName = 'hive_encryption_key';

  static Future<void> init() async {
    await Hive.initFlutter();

    // Register type adapters
    Hive.registerAdapter(TaskAdapter());
    Hive.registerAdapter(TaskStatusAdapter());
    Hive.registerAdapter(ProfileAdapter());

    // Open plain (non-sensitive) boxes
    await Hive.openBox(_settingsBox);

    // Open encrypted boxes for sensitive cached data
    final cipher = await _resolveCipher();
    await Hive.openBox<Profile>(_userProfileBox, encryptionCipher: cipher);
    await Hive.openBox<Task>(_taskCacheBox, encryptionCipher: cipher);
  }

  static Future<HiveAesCipher> _resolveCipher() async {
    const secureStorage = FlutterSecureStorage();
    String? base64Key = await secureStorage.read(key: _encryptionKeyName);
    if (base64Key == null) {
      final key = Hive.generateSecureKey();
      base64Key = base64Encode(key);
      await secureStorage.write(key: _encryptionKeyName, value: base64Key);
    }
    return HiveAesCipher(base64Decode(base64Key));
  }

  /// Clears all cached boxes (call on logout)
  static Future<void> clearUserData() async {
    await Hive.box<Profile>(_userProfileBox).clear();
    await Hive.box<Task>(_taskCacheBox).clear();
  }
}
```

### Typed Box Providers

```dart
// lib/core/local_storage/box_provider.dart
import 'package:hive/hive.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../features/profile/domain/profile.dart';
import '../../features/task/domain/task.dart';

part 'box_provider.g.dart';

@Riverpod(keepAlive: true)
Box settingsBox(SettingsBoxRef ref) => Hive.box('settings');

@Riverpod(keepAlive: true)
Box<Profile> profileBox(ProfileBoxRef ref) => Hive.box<Profile>('user_profile');

@Riverpod(keepAlive: true)
Box<Task> taskCacheBox(TaskCacheBoxRef ref) => Hive.box<Task>('task_cache');
```

### Secure Storage Provider

```dart
// lib/core/local_storage/secure_storage_provider.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'secure_storage_provider.g.dart';

@Riverpod(keepAlive: true)
FlutterSecureStorage secureStorage(SecureStorageRef ref) {
  return const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );
}
```

---

## Hive Type Adapter (Co-located With Freezed Model)

Annotate freezed model fields with `@HiveField(N)` to generate adapters. The Hive
generator picks them up via `build_runner`:

```dart
// lib/features/task/domain/task.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:hive/hive.dart';

part 'task.freezed.dart';
part 'task.g.dart';

@HiveType(typeId: 1)
enum TaskStatus {
  @HiveField(0) @JsonValue('DRAFT')       draft,
  @HiveField(1) @JsonValue('IN_PROGRESS') inProgress,
  @HiveField(2) @JsonValue('COMPLETED')   completed,
}

@freezed
class Task with _$Task {
  @HiveType(typeId: 2)
  const factory Task({
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

> Each module's `typeId` must be unique across the application — keep a registry in a
> central `lib/core/local_storage/type_ids.dart` if many modules use Hive.

---

## Cache-Aside Read Pattern

```dart
// lib/features/task/data/task_repository.dart (excerpt)
class TaskRepository {
  TaskRepository(this._dio, this._cacheBox);
  final Dio _dio;
  final Box<Task> _cacheBox;

  Future<List<Task>> listCached() async {
    return _cacheBox.values.toList();
  }

  Future<List<Task>> refreshList(TaskListParams params) async {
    try {
      final response = await _dio.get<Map<String, dynamic>>(
        '/tasks',
        queryParameters: params.toJson(),
      );
      final page = PaginatedResponse.fromJson(
        response.data!,
        (json) => Task.fromJson(json as Map<String, dynamic>),
      );
      // Replace cache with fresh page
      await _cacheBox.clear();
      await _cacheBox.addAll(page.content);
      return page.content;
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }

  Future<void> upsertCached(Task task) async {
    final key = _cacheBox.values.toList().indexWhere((t) => t.id == task.id);
    if (key >= 0) {
      await _cacheBox.putAt(key, task);
    } else {
      await _cacheBox.add(task);
    }
  }
}
```

---

## Paginated Response Type

```dart
// lib/core/api/paginated_response.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'paginated_response.freezed.dart';

@Freezed(genericArgumentFactories: true)
class PaginatedResponse<T> with _$PaginatedResponse<T> {
  const factory PaginatedResponse({
    required List<T> content,
    required int page,
    required int size,
    required int totalElements,
    required int totalPages,
  }) = _PaginatedResponse<T>;

  factory PaginatedResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object?) fromJsonT,
  ) {
    return PaginatedResponse<T>(
      content: (json['content'] as List).map((e) => fromJsonT(e)).toList(),
      page: json['page'] as int,
      size: json['size'] as int,
      totalElements: json['totalElements'] as int,
      totalPages: json['totalPages'] as int,
    );
  }
}
```

---

## Settings & Preferences

Use the plain `settings` box for simple key-value preferences:

```dart
// Theme mode
ref.read(settingsBoxProvider).put('themeMode', 'dark');
final mode = ref.read(settingsBoxProvider).get('themeMode') as String?;

// Locale
ref.read(settingsBoxProvider).put('locale', 'ms');

// First-launch flag
final firstLaunch = !(ref.read(settingsBoxProvider).get('hasLaunched') as bool? ?? false);
await ref.read(settingsBoxProvider).put('hasLaunched', true);
```

For complex typed preferences, define a small `@freezed` class with a Hive adapter and
store it in a dedicated box (e.g., `Box<UserPreferences>`).

---

## Dio Configuration (Recap)

The Dio instance is constructed once in `dioProvider` and shared via Riverpod. Its
interceptors are layered in this exact order:

1. **AuthInterceptor** — injects Bearer token (reads via `AuthRepository.getAccessToken`)
2. **RetryInterceptor (dio_smart_retry)** — exponential backoff on 5xx and network
   errors; only retries idempotent methods (GET, PUT, DELETE)
3. **PrettyDioLogger** — verbose request/response logging in `kDebugMode` only

```dart
// (Already shown in spec-template Section 8)
```

### `pretty_dio_logger` Configuration

```dart
PrettyDioLogger(
  requestHeader: true,    // log headers
  requestBody: true,      // log request body
  responseBody: true,     // log response body
  responseHeader: false,
  error: true,
  compact: false,
  maxWidth: 90,
);
```

### Retry Interceptor

```dart
RetryInterceptor(
  dio: dio,
  retries: 3,
  retryDelays: const [
    Duration(seconds: 1),
    Duration(seconds: 2),
    Duration(seconds: 4),
  ],
  retryableExtraStatuses: const {502, 503, 504},
  // By default, dio_smart_retry only retries idempotent methods (GET, HEAD, PUT,
  // DELETE, OPTIONS, TRACE). POST is NOT retried by default — which is correct for
  // most mutations. Override per-request via `RequestOptions.extra['retry'] = true`
  // for explicitly idempotent POSTs.
);
```

### Per-Request Retry Override

To enable retries for a single (idempotent-by-design) POST:

```dart
await dio.post(
  '/audit-logs',
  data: payload,
  options: Options(extra: {RetryInterceptor.kRetryAttemptsKey: 3}),
);
```

---

## Error Mapping at the Boundary

Repositories convert `DioException` into the sealed `ApiFailure` type so that screens
deal with a typed failure rather than transport-level details:

```dart
try {
  final response = await _dio.get<Map<String, dynamic>>('/tasks/$id');
  return Task.fromJson(response.data!);
} on DioException catch (e) {
  throw ApiFailure.fromDioException(e);
}
```

`ApiFailure.fromDioException` (see `spec-template.md` Section 8) handles:
- Network timeouts → `ApiFailure.network()`
- 401 → `ApiFailure.unauthorized()`
- 403 → `ApiFailure.forbidden()`
- 404 → `ApiFailure.notFound()`
- 5xx → `ApiFailure.server(status, message)`
- Everything else → `ApiFailure.unknown(error)`

Screens consume failures via Riverpod `AsyncValue.when(error: ...)` and feed them into
the `ErrorState` widget for consistent UX.

---

## Migration Strategy

When a freezed/Hive model adds a field with a new `@HiveField` index:
- The new field MUST be nullable or have a default value — Hive treats missing fields
  as `null` for older records.
- Never reuse a removed `@HiveField` index. Mark the field with `@Deprecated` if you
  need to keep reading it; otherwise remove and never assign that index again.
- For breaking schema changes (renamed type, restructured fields), bump the box name
  (e.g., `task_cache` → `task_cache_v2`) and run a one-time clear-and-refetch on startup:

```dart
final box = await Hive.openBox<Task>('task_cache_v2');
if (Hive.boxExists('task_cache')) {
  await Hive.deleteBoxFromDisk('task_cache');
}
```

---

## Logout Hygiene

On user logout, clear all caches and secure-storage keys to avoid leaking data to the
next user on a shared device:

```dart
Future<void> performLogout(WidgetRef ref) async {
  await ref.read(authProvider.notifier).signOut();
  await HiveSetup.clearUserData();
  // Secure storage keys cleared inside AuthRepository.signOutLocal()
}
```
