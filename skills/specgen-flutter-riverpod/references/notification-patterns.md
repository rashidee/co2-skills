# Notification Patterns — FCM + flutter_local_notifications

This reference describes how Firebase Cloud Messaging (push) and
`flutter_local_notifications` (local/foreground/scheduled) are wired together. Include
this content in Section 16 of the generated specification.

---

## Why Both Libraries?

- **Firebase Messaging (FCM)** delivers messages from the server. It shows system
  notifications automatically when the app is **backgrounded** or **terminated** (data
  + notification payload).
- **Foreground delivery does NOT auto-display.** On Android and iOS, FCM messages
  received while the app is in the foreground only fire the `onMessage` callback —
  the developer must render a notification (or in-app UI) themselves.
- **`flutter_local_notifications`** is the rendering library for those foreground FCM
  payloads and for any in-app scheduled reminders (alarms, follow-ups).

---

## Initialization

```dart
// lib/core/notification/notification_service.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

import 'notification_handler.dart';

class NotificationService {
  NotificationService._();
  static final NotificationService instance = NotificationService._();

  final FlutterLocalNotificationsPlugin _local = FlutterLocalNotificationsPlugin();
  final FirebaseMessaging _fcm = FirebaseMessaging.instance;

  /// Android notification channel for default notifications. Must match the channel
  /// id referenced in AndroidManifest.xml (`default_notification_channel_id`).
  static const AndroidNotificationChannel _defaultChannel = AndroidNotificationChannel(
    'default',
    'General Notifications',
    description: 'Default channel for app notifications',
    importance: Importance.high,
  );

  Future<void> init() async {
    await _initLocal();
    await _initFcm();
  }

  Future<void> _initLocal() async {
    const androidInit = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosInit = DarwinInitializationSettings(
      requestAlertPermission: false, // request explicitly via permission flow
      requestBadgePermission: false,
      requestSoundPermission: false,
    );
    const initSettings = InitializationSettings(android: androidInit, iOS: iosInit);

    await _local.initialize(
      initSettings,
      onDidReceiveNotificationResponse: NotificationHandler.onLocalTap,
    );

    final androidPlugin =
        _local.resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>();
    await androidPlugin?.createNotificationChannel(_defaultChannel);
  }

  Future<void> _initFcm() async {
    // Request permission (iOS + Android 13+)
    await _fcm.requestPermission(
      alert: true,
      announcement: false,
      badge: true,
      carPlay: false,
      criticalAlert: false,
      provisional: false,
      sound: true,
    );

    // Foreground presentation options for iOS (banner + sound while app is open)
    await _fcm.setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );

    // Foreground message handler — show as local notification
    FirebaseMessaging.onMessage.listen(NotificationHandler.onForegroundMessage);

    // Notification tap (from background → foreground)
    FirebaseMessaging.onMessageOpenedApp.listen(NotificationHandler.onMessageOpenedApp);

    // Cold start (terminated → foreground via notification tap)
    final initialMessage = await _fcm.getInitialMessage();
    if (initialMessage != null) {
      // Defer until router is mounted
      WidgetsBinding.instance.addPostFrameCallback((_) {
        NotificationHandler.onMessageOpenedApp(initialMessage);
      });
    }
  }

  Future<String?> fcmToken() => _fcm.getToken();

  Stream<String> onTokenRefresh() => _fcm.onTokenRefresh;

  /// Subscribe to a topic (e.g., for role-based broadcasts)
  Future<void> subscribeToTopic(String topic) => _fcm.subscribeToTopic(topic);

  Future<void> unsubscribeFromTopic(String topic) => _fcm.unsubscribeFromTopic(topic);

  /// Show a local notification immediately. Used to display foreground FCM payloads.
  Future<void> showLocal({
    required String title,
    required String body,
    String? payload,
  }) async {
    const details = NotificationDetails(
      android: AndroidNotificationDetails(
        'default',
        'General Notifications',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      ),
    );
    await _local.show(
      DateTime.now().millisecondsSinceEpoch.remainder(1 << 31),
      title,
      body,
      details,
      payload: payload,
    );
  }

  /// Schedule a local notification at a specific time (e.g., reminder).
  Future<void> schedule({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledAt,
    String? payload,
  }) async {
    const details = NotificationDetails(
      android: AndroidNotificationDetails(
        'default',
        'General Notifications',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: DarwinNotificationDetails(),
    );
    // Note: `tz.TZDateTime` from package:timezone is preferred; for brevity use
    // local DateTime here. In a production app, initialise timezones in main().
    await _local.zonedSchedule(
      id,
      title,
      body,
      // Convert to tz.TZDateTime.from(scheduledAt, tz.local) in production
      scheduledAt as dynamic,
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      payload: payload,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  Future<void> cancel(int id) => _local.cancel(id);
  Future<void> cancelAll() => _local.cancelAll();
}
```

---

## Handlers

```dart
// lib/core/notification/notification_handler.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

import 'notification_navigation.dart';
import 'notification_service.dart';

class NotificationHandler {
  NotificationHandler._();

  /// Called when an FCM message is received while the app is in the foreground.
  /// FCM does NOT auto-display these — render a local notification instead.
  static Future<void> onForegroundMessage(RemoteMessage message) async {
    final notification = message.notification;
    if (notification == null) return;

    await NotificationService.instance.showLocal(
      title: notification.title ?? '',
      body: notification.body ?? '',
      payload: message.data['deeplink'] as String?,
    );
  }

  /// Called when the user taps an FCM notification (background or terminated).
  static Future<void> onMessageOpenedApp(RemoteMessage message) async {
    final deeplink = message.data['deeplink'] as String?;
    if (deeplink != null) {
      NotificationNavigator.go(deeplink);
    }
  }

  /// Called when the user taps a local notification (in-app reminders or
  /// foreground-rendered FCM messages).
  static void onLocalTap(NotificationResponse response) {
    final deeplink = response.payload;
    if (deeplink != null && deeplink.isNotEmpty) {
      NotificationNavigator.go(deeplink);
    }
  }
}
```

---

## Background Message Handler (Top-level)

The background handler MUST be a top-level (not class member) function annotated with
`@pragma('vm:entry-point')` so it survives release-mode tree shaking. It is registered
in `main.dart` (already shown):

```dart
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  // No UI work here — the system displays the notification automatically when the
  // payload includes a `notification:` block. Only do background data processing.
}
```

---

## Deep-Link Navigation From Notifications

The router lives inside a Riverpod provider, so we expose the root `ProviderContainer`
in `app.dart` and reach it from outside the widget tree:

```dart
// lib/core/notification/notification_navigation.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../router/app_router.dart';

class NotificationNavigator {
  NotificationNavigator._();

  /// Root container set by the app on startup.
  static ProviderContainer? container;

  static void go(String location) {
    final c = container;
    if (c == null) return;
    final router = c.read(appRouterProvider);
    router.go(location);
  }
}
```

In `main.dart`, expose the container:

```dart
void main() async {
  // ...existing init...
  final container = ProviderContainer();
  NotificationNavigator.container = container;
  runApp(
    UncontrolledProviderScope(container: container, child: const MyApp()),
  );
}
```

---

## Backend Token Registration

Register the FCM token with the backend on first sign-in and on token refresh:

```dart
// inside auth flow, after successful login:
final token = await NotificationService.instance.fcmToken();
if (token != null) {
  await dio.post('/users/me/fcm-token', data: {'token': token});
}

NotificationService.instance.onTokenRefresh().listen((newToken) async {
  await dio.post('/users/me/fcm-token', data: {'token': newToken});
});
```

Unregister the token on logout:

```dart
await dio.delete('/users/me/fcm-token');
await FirebaseMessaging.instance.deleteToken();
```

---

## Permission Handling

| Platform | Behaviour |
|---|---|
| iOS | `FirebaseMessaging.requestPermission()` triggers the system prompt |
| Android 13+ | `POST_NOTIFICATIONS` runtime permission required |
| Android <13 | Notifications enabled by default |

```dart
// Example Android 13+ permission flow using permission_handler (if added):
import 'package:permission_handler/permission_handler.dart';

final status = await Permission.notification.request();
if (status.isDenied) {
  // Surface a settings prompt
}
```

---

## Notification Payload Contract

The backend should send messages in this shape so the client can route consistently:

```json
{
  "notification": {
    "title": "New task assigned",
    "body": "Inspect site #42"
  },
  "data": {
    "deeplink": "/task/abc-123",
    "category": "task"
  }
}
```

`deeplink` is read by `NotificationHandler.onMessageOpenedApp` and passed to
`router.go(...)`.

---

## Topic Subscription Strategy

Topics are useful for role/segment broadcasts (e.g., all workers, all admins):

```dart
// On login, subscribe to role-based topics
for (final role in user.roles) {
  await NotificationService.instance.subscribeToTopic('role-${role.toLowerCase()}');
}

// On logout, unsubscribe
for (final role in user.roles) {
  await NotificationService.instance.unsubscribeFromTopic('role-${role.toLowerCase()}');
}
```

---

## Scheduled Local Reminders

Use `zonedSchedule` for time-based reminders (e.g., "Inspect site at 09:00 tomorrow"):

```dart
await NotificationService.instance.schedule(
  id: task.hashCode,
  title: 'Task reminder',
  body: task.title,
  scheduledAt: task.dueDate.subtract(const Duration(hours: 1)),
  payload: '/task/${task.id}',
);
```

For production, initialise the `timezone` package in `main()`:

```dart
// pubspec.yaml → dependencies: timezone: ^0.9.4
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

tz.initializeTimeZones();
tz.setLocalLocation(tz.getLocation('Asia/Kuala_Lumpur')); // or detect from device
```

---

## iOS-Specific Notes

- Add the **Push Notifications** capability in Xcode (Runner target → Signing &
  Capabilities)
- Add the **Background Modes → Remote notifications** capability
- Upload an **APNs Authentication Key** to Firebase Console → Project Settings → Cloud
  Messaging
- `Info.plist` must include `NSUserNotificationUsageDescription` if you display
  notifications while the device is locked

---

## Android-Specific Notes

- The Android `default` channel MUST be created on app startup (done in
  `_initLocal`). Otherwise high-importance notifications fall back to silent on
  Android 8.0+.
- `AndroidManifest.xml` includes the channel meta-data so FCM uses the correct channel
  for background-delivered notifications:

  ```xml
  <meta-data
    android:name="com.google.firebase.messaging.default_notification_channel_id"
    android:value="default" />
  ```

- Add the Android 13+ `POST_NOTIFICATIONS` permission to `AndroidManifest.xml`:

  ```xml
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
  ```

---

## Testing Notifications Locally

- **FCM from console**: Firebase Console → Cloud Messaging → New campaign → Test on
  device using FCM token printed at startup
- **Local foreground**: trigger `NotificationService.instance.showLocal(...)` from a
  debug button
- **Scheduled local**: short-delay schedule (`DateTime.now().add(Duration(seconds: 5))`)
  to verify channel and tap routing

---

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Foreground FCM never shown | You MUST manually render a local notification — FCM does not auto-display |
| Android 13+ silent | Request `POST_NOTIFICATIONS` runtime permission |
| iOS background not delivered | Ensure APNs key uploaded to Firebase, Background Modes enabled |
| Deep-link doesn't navigate on cold start | Read `getInitialMessage()` AFTER router mounts (use `addPostFrameCallback`) |
| FCM token churns after reinstall | Re-register on every app launch, deduplicate server-side |
| Scheduled notifications not firing | Confirm `androidScheduleMode: exactAllowWhileIdle` and request `SCHEDULE_EXACT_ALARM` on Android 12+ |
