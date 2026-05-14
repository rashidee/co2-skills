# State Management Patterns — Riverpod 2 with `riverpod_generator`

This reference describes the canonical Riverpod patterns for the spec. Include this
content in Section 10 of the generated specification and in any module SPEC.md that
defines providers.

---

## State Separation Principle

The golden rule for Flutter app state management:

| State Type | Tool | Examples |
|---|---|---|
| **Server state** (data from API) | `AsyncNotifier` / `FutureProvider` | Task list, profile, catalogue |
| **Cached server state** (offline) | Hive box + Riverpod provider | Last-seen tasks, profile snapshot |
| **UI state** (per-screen, transient) | `Notifier` / `useState` (flutter_hooks) | Selected filter, active tab |
| **Global UI state** | `Notifier` persisted to Hive | Theme mode, locale, sidebar |
| **Auth state** | `AsyncNotifier`; tokens in `flutter_secure_storage` | JWT, user info, roles |
| **Form state** | `Form` + controllers (+ optional `Notifier` for wizards) | Field values, validation errors |

**Do NOT cache API responses in plain `Notifier`/`StateProvider`.** Use `AsyncNotifier`
so that loading/error/data states are first-class. For offline persistence, write the
fetched data to Hive *inside* the provider's `build()` after a successful fetch, and
seed `build()` with the cached value when offline.

---

## Generated Providers — `build.yaml`

```yaml
# build.yaml at the project root
targets:
  $default:
    builders:
      riverpod_generator:
        options:
          # Provide a project-wide naming convention if desired
          provider_family_name: family
```

Every provider file imports the generator part:

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'task_providers.g.dart';
```

Run codegen with:
```bash
dart run build_runner watch --delete-conflicting-outputs
```

---

## Repository Provider Pattern

```dart
// lib/features/task/data/task_repository.dart
import 'package:dio/dio.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../core/api/api_failure.dart';
import '../../../core/api/dio_provider.dart';
import '../../../core/api/paginated_response.dart';
import '../domain/task.dart';

part 'task_repository.g.dart';

class TaskRepository {
  TaskRepository(this._dio);
  final Dio _dio;

  Future<PaginatedResponse<Task>> list(TaskListParams params) async {
    try {
      final response = await _dio.get<Map<String, dynamic>>(
        '/tasks',
        queryParameters: params.toJson(),
      );
      return PaginatedResponse.fromJson(
        response.data!,
        (item) => Task.fromJson(item as Map<String, dynamic>),
      );
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }

  Future<Task> getById(String id) async {
    try {
      final response = await _dio.get<Map<String, dynamic>>('/tasks/$id');
      return Task.fromJson(response.data!);
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }

  Future<Task> create(TaskCreate payload) async {
    try {
      final response = await _dio.post<Map<String, dynamic>>(
        '/tasks',
        data: payload.toJson(),
      );
      return Task.fromJson(response.data!);
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }

  Future<Task> update(String id, TaskUpdate payload) async {
    try {
      final response = await _dio.put<Map<String, dynamic>>(
        '/tasks/$id',
        data: payload.toJson(),
      );
      return Task.fromJson(response.data!);
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }

  Future<void> delete(String id) async {
    try {
      await _dio.delete<void>('/tasks/$id');
    } on DioException catch (e) {
      throw ApiFailure.fromDioException(e);
    }
  }
}

@Riverpod(keepAlive: true)
TaskRepository taskRepository(TaskRepositoryRef ref) {
  return TaskRepository(ref.watch(dioProvider));
}
```

---

## AsyncNotifier (List) Pattern

```dart
// lib/features/task/application/task_providers.dart
import 'package:flutter/foundation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../data/task_repository.dart';
import '../domain/task.dart';

part 'task_providers.g.dart';

@immutable
class TaskListParams {
  const TaskListParams({
    this.page = 0,
    this.size = 20,
    this.status,
    this.search,
  });

  final int page;
  final int size;
  final TaskStatus? status;
  final String? search;

  Map<String, dynamic> toJson() => {
    'page': page,
    'size': size,
    if (status != null) 'status': status!.name.toUpperCase(),
    if (search != null && search!.isNotEmpty) 'search': search,
  };

  TaskListParams copyWith({int? page, int? size, TaskStatus? status, String? search}) {
    return TaskListParams(
      page: page ?? this.page,
      size: size ?? this.size,
      status: status ?? this.status,
      search: search ?? this.search,
    );
  }

  @override
  bool operator ==(Object other) =>
      other is TaskListParams &&
      other.page == page &&
      other.size == size &&
      other.status == status &&
      other.search == search;

  @override
  int get hashCode => Object.hash(page, size, status, search);
}

@riverpod
class TaskList extends _$TaskList {
  @override
  Future<PaginatedResponse<Task>> build(TaskListParams params) async {
    final repo = ref.watch(taskRepositoryProvider);
    return repo.list(params);
  }

  Future<void> refresh() async {
    ref.invalidateSelf();
    await future;
  }
}

@riverpod
Future<Task> taskById(TaskByIdRef ref, String id) async {
  final repo = ref.watch(taskRepositoryProvider);
  return repo.getById(id);
}
```

---

## Mutation Notifier Pattern

```dart
@riverpod
class TaskFormController extends _$TaskFormController {
  @override
  AsyncValue<void> build() => const AsyncData(null);

  Future<Task> create(TaskCreate payload) async {
    state = const AsyncLoading();
    try {
      final repo = ref.read(taskRepositoryProvider);
      final created = await repo.create(payload);
      ref.invalidate(taskListProvider);
      state = const AsyncData(null);
      return created;
    } catch (error, stack) {
      state = AsyncError(error, stack);
      rethrow;
    }
  }

  Future<Task> update(String id, TaskUpdate payload) async {
    state = const AsyncLoading();
    try {
      final repo = ref.read(taskRepositoryProvider);
      final updated = await repo.update(id, payload);
      ref.invalidate(taskListProvider);
      ref.invalidate(taskByIdProvider(id));
      state = const AsyncData(null);
      return updated;
    } catch (error, stack) {
      state = AsyncError(error, stack);
      rethrow;
    }
  }

  Future<void> delete(String id) async {
    state = const AsyncLoading();
    try {
      final repo = ref.read(taskRepositoryProvider);
      await repo.delete(id);
      ref.invalidate(taskListProvider);
      state = const AsyncData(null);
    } catch (error, stack) {
      state = AsyncError(error, stack);
      rethrow;
    }
  }
}
```

---

## Consuming Providers in Widgets

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../application/task_providers.dart';
import '../../../shared/widgets/empty_state.dart';
import '../../../shared/widgets/error_state.dart';
import '../../../shared/widgets/loading_indicator.dart';

class TaskListScreen extends ConsumerStatefulWidget {
  const TaskListScreen({super.key});

  @override
  ConsumerState<TaskListScreen> createState() => _TaskListScreenState();
}

class _TaskListScreenState extends ConsumerState<TaskListScreen> {
  TaskListParams _params = const TaskListParams();

  @override
  Widget build(BuildContext context) {
    final taskListAsync = ref.watch(taskListProvider(_params));

    return Scaffold(
      appBar: AppBar(title: const Text('Tasks')),
      body: taskListAsync.when(
        loading: () => const LoadingIndicator(),
        error: (error, _) => ErrorState(
          error: error,
          onRetry: () => ref.invalidate(taskListProvider(_params)),
        ),
        data: (page) {
          if (page.content.isEmpty) {
            return const EmptyState(message: 'No tasks found.');
          }
          return ListView.builder(
            itemCount: page.content.length,
            itemBuilder: (context, index) {
              final task = page.content[index];
              return ListTile(
                title: Text(task.title),
                subtitle: Text(task.status.name),
                onTap: () => context.push('/task/${task.id}'),
              );
            },
          );
        },
      ),
    );
  }
}
```

---

## Offline-First Pattern (Hive cache + remote)

```dart
@riverpod
class TaskList extends _$TaskList {
  @override
  Future<List<Task>> build(TaskListParams params) async {
    final box = ref.watch(taskCacheBoxProvider);
    final repo = ref.watch(taskRepositoryProvider);

    // Seed with cached data if present (sync)
    final cached = box.values.toList();

    try {
      final remote = await repo.list(params);
      await box.clear();
      await box.addAll(remote.content);
      return remote.content;
    } catch (e) {
      // Surface fresh data when possible, but fall back to cache
      if (cached.isNotEmpty) {
        return cached;
      }
      rethrow;
    }
  }
}
```

---

## Family Providers for Detail / Parametrised Queries

```dart
@riverpod
Future<Task> taskById(TaskByIdRef ref, String id) async {
  final repo = ref.watch(taskRepositoryProvider);
  return repo.getById(id);
}

// Usage in widget:
//   final taskAsync = ref.watch(taskByIdProvider('abc-123'));
```

---

## Listening for Side Effects (`ref.listen`)

Use `ref.listen` to react to provider state changes without rebuilding the widget —
useful for navigation, SnackBars, and dialogs:

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  ref.listen<AsyncValue<void>>(taskFormControllerProvider, (previous, next) {
    next.whenOrNull(
      error: (error, _) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(error.toString())),
        );
      },
    );
  });
  // ...
}
```

---

## Multi-Step Form / Wizard Pattern

```dart
@freezed
class TaskWizardState with _$TaskWizardState {
  const factory TaskWizardState({
    @Default(0) int step,
    @Default(null) TaskCreateStep1? step1,
    @Default(null) TaskCreateStep2? step2,
  }) = _TaskWizardState;
}

@riverpod
class TaskWizard extends _$TaskWizard {
  @override
  TaskWizardState build() => const TaskWizardState();

  void saveStep1(TaskCreateStep1 data) =>
      state = state.copyWith(step1: data, step: 1);

  void saveStep2(TaskCreateStep2 data) =>
      state = state.copyWith(step2: data, step: 2);

  void back() => state = state.copyWith(step: state.step - 1);

  void reset() => state = const TaskWizardState();
}
```

---

## Cache Invalidation Strategy

Follow this decision tree when invalidating Riverpod state after mutations:

| Mutation | Invalidation Target |
|---|---|
| Create item | `ref.invalidate(<entity>ListProvider)` |
| Update item | `ref.invalidate(<entity>ListProvider)` + `ref.invalidate(<entity>ByIdProvider(id))` |
| Delete item | `ref.invalidate(<entity>ListProvider)` |
| Bulk action (delete multiple) | `ref.invalidate(<entity>ListProvider)` and individual `<entity>ByIdProvider(id)` |
| Status toggle | `ref.invalidate(<entity>ListProvider)` + `ref.invalidate(<entity>ByIdProvider(id))` |

Prefer `ref.invalidate(...)` over manually mutating provider state — invalidation lets
the next `watch` rebuild fresh from source while preserving the loading UI.

---

## Auth State Provider

```dart
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.unauthenticated() = _Unauthenticated;
  const factory AuthState.authenticated({
    required AuthUser user,
    required String accessToken,
  }) = _Authenticated;
}

@Riverpod(keepAlive: true)
class Auth extends _$Auth {
  @override
  FutureOr<AuthState> build() async {
    final repo = ref.watch(authRepositoryProvider);
    final restored = await repo.restoreSession();
    return restored ?? const AuthState.unauthenticated();
  }

  Future<void> login(String email, String password) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(authRepositoryProvider);
      final session = await repo.login(email, password);
      return AuthState.authenticated(
        user: session.user,
        accessToken: session.accessToken,
      );
    });
  }

  Future<void> logout() async {
    final repo = ref.read(authRepositoryProvider);
    await repo.logout();
    state = const AsyncData(AuthState.unauthenticated());
  }
}
```

---

## Testing Providers in Isolation

```dart
// test/features/task/task_list_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:mocktail/mocktail.dart';

import 'package:mobile_app/features/task/data/task_repository.dart';
import 'package:mobile_app/features/task/application/task_providers.dart';

class _MockTaskRepository extends Mock implements TaskRepository {}

void main() {
  test('taskList returns paginated tasks', () async {
    final repo = _MockTaskRepository();
    when(() => repo.list(any())).thenAnswer((_) async => PaginatedResponse(
      content: [const Task(id: '1', title: 'A', status: TaskStatus.draft, /*...*/)],
      page: 0, size: 20, totalElements: 1, totalPages: 1,
    ));

    final container = ProviderContainer(overrides: [
      taskRepositoryProvider.overrideWithValue(repo),
    ]);
    addTearDown(container.dispose);

    final page = await container.read(taskListProvider(const TaskListParams()).future);
    expect(page.content, hasLength(1));
  });
}
```
