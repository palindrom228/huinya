---
type: question
category: android-core
difficulty: middle
tags: [notifications, channels, fcm]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Notifications: channels, importance, FCM, POST_NOTIFICATIONS, inline reply

## Краткий ответ (TL;DR)

С Android 8+ каждое уведомление **обязано** принадлежать **Notification Channel** (создаётся в коде, юзер контролирует importance через системные настройки). С Android 13+ нужно runtime permission `POST_NOTIFICATIONS`. Для push — Firebase Cloud Messaging (FCM); high-priority push могут запускать ваш код в фоне.

## Развёрнутый ответ

### Notification Channels (8.0+)

```kotlin
val mgr = NotificationManagerCompat.from(context)
mgr.createNotificationChannel(
    NotificationChannelCompat.Builder("orders", IMPORTANCE_HIGH)
        .setName("Orders")
        .setDescription("New order alerts")
        .setVibrationEnabled(true)
        .build()
)
```

Importance уровни:
- `NONE` — не показываются.
- `MIN` — silent, нет в status bar.
- `LOW` — в drawer, без звука.
- `DEFAULT` — звук, без heads-up.
- `HIGH` — звук + heads-up (поп-ап).

Пользователь может **поменять** importance через настройки — приложение не может перебить.

### Notification

```kotlin
val notif = NotificationCompat.Builder(context, "orders")
    .setSmallIcon(R.drawable.ic_order)
    .setContentTitle("New order #${id}")
    .setContentText("Tap to view")
    .setStyle(NotificationCompat.BigTextStyle().bigText(longText))
    .setContentIntent(PendingIntent.getActivity(context, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE))
    .addAction(replyAction)
    .setAutoCancel(true)
    .build()

mgr.notify(NOTIF_ID, notif)
```

### POST_NOTIFICATIONS (Android 13+)

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```kotlin
val launcher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted -> /* ... */ }

if (Build.VERSION.SDK_INT >= 33 &&
    ContextCompat.checkSelfPermission(this, POST_NOTIFICATIONS) != GRANTED) {
    launcher.launch(POST_NOTIFICATIONS)
}
```

Запрашивать **в моменте, когда уместно** (например, после онбординга), а не на splash.

### Heads-up notifications

Появляются только если:
- Channel importance ≥ HIGH.
- Имеют `setFullScreenIntent` (для inbox/звонков) — требует permission `USE_FULL_SCREEN_INTENT` (Android 14+).
- Не в Do Not Disturb (если не bypass).

### Action buttons + inline reply

```kotlin
val remoteInput = RemoteInput.Builder("reply_key")
    .setLabel("Reply")
    .build()

val replyPi = PendingIntent.getBroadcast(context, 0, replyIntent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE)  // mutable обязателен

val replyAction = NotificationCompat.Action.Builder(R.drawable.ic_reply, "Reply", replyPi)
    .addRemoteInput(remoteInput)
    .setSemanticAction(SEMANTIC_ACTION_REPLY)
    .build()
```

В Receiver:

```kotlin
val text = RemoteInput.getResultsFromIntent(intent)?.getCharSequence("reply_key")
```

### FCM

```kotlin
class MyFcmService : FirebaseMessagingService() {
    override fun onNewToken(token: String) { sendToBackend(token) }
    override fun onMessageReceived(msg: RemoteMessage) {
        // notification payload → системой обработано (если app в background и есть notification block)
        // data payload → всегда сюда
    }
}
```

#### Priority

- **Normal** — может задержаться (Doze).
- **High** — мгновенная доставка, открывает app для обработки.
- На Android 12+ для FGS из push нужен либо exemption, либо expedited WorkManager.

### Notification trampolines (запрет с Android 12+)

```kotlin
// ❌ запрещено: notification → BroadcastReceiver → startActivity
```

Notification должен **сразу** запускать Activity (через `setContentIntent`), а не пробрасывать через Service/Receiver. Иначе системой блокируется.

### Foreground service notification

Foreground service всегда показывает уведомление. С Android 13+ можно «свернуть» его, но скрыть полностью нельзя для большинства типов.

## Подводные камни

- Channel создан с дефолтным importance → его уже нельзя поднять кодом (только пересоздать с новым id).
- На Android 13+ без `POST_NOTIFICATIONS` уведомления не показываются.
- FCM data-only payload в **killed** state приложения → onMessageReceived вызовется (но процесс холодно стартует).
- Inline reply без `FLAG_MUTABLE` упадёт с exception на Android 12+.
- Notification trampoline через Receiver → activity не запустится (silent fail).

## Связанные темы

- [[q-04-intents]]
- [[q-11-permissions]]
- [[q-06-services]]

## Follow-up

- Почему importance канала нельзя повысить после создания?
- Чем отличаются notification и data payloads в FCM?
- Что такое notification trampoline и почему его запретили?
