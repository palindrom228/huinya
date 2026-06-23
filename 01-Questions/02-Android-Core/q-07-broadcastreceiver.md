---
type: question
category: android-core
difficulty: middle
tags: [broadcastreceiver, manifest]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# BroadcastReceiver: manifest vs context-registered, ordered, local. Ограничения Oreo+.

## Краткий ответ (TL;DR)

**Manifest-registered** — статический, ловит system broadcasts даже при не запущенном приложении. **Context-registered** — динамический, работает пока зарегистрирован. С Android 8.0 **большинство implicit broadcasts** запрещены в manifest (нужно регистрировать динамически). LocalBroadcastManager **deprecated** — используй `Flow`/`LiveData`/`EventBus` внутри процесса.

## Развёрнутый ответ

### Manifest-registered

```xml
<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

Система разбудит процесс, создаст экземпляр, вызовет `onReceive`. Доступно ограниченное время (~10 сек) — для долгой работы вызывай `goAsync()` или планируй WorkManager.

#### Android 8.0 ограничения

Implicit broadcasts (без явного component) **нельзя** регистрировать через манифест. Исключения: ограниченный whitelist (BOOT_COMPLETED, LOCKED_BOOT_COMPLETED, TIME_SET, PACKAGE_FULLY_REMOVED, и т.д.).

Полный список: [official docs](https://developer.android.com/guide/components/broadcast-exceptions).

### Context-registered

```kotlin
val receiver = MyReceiver()
val filter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
context.registerReceiver(receiver, filter)

// важно отписаться
context.unregisterReceiver(receiver)
```

С Android 13+ обязателен `RECEIVER_EXPORTED`/`RECEIVER_NOT_EXPORTED` для незащищённых broadcast'ов:

```kotlin
ContextCompat.registerReceiver(
    context, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED
)
```

### goAsync

```kotlin
override fun onReceive(context: Context, intent: Intent) {
    val pending = goAsync()
    scope.launch {
        try { doWork() } finally { pending.finish() }
    }
}
```

Расширяет окно жизни receiver до ~10–30 сек. Но не панацея: процесс могут убить.

### Ordered broadcasts

```kotlin
context.sendOrderedBroadcast(intent, permission)
```

Receivers вызываются по приоритету (`android:priority`), могут модифицировать результат или прервать (`abortBroadcast`). Используется редко (например, SMS receive).

### Sticky broadcasts — deprecated

`sendStickyBroadcast` устарел. Использовать LiveData/StateFlow вместо.

### LocalBroadcastManager

Deprecated. Замена:
- Внутрипроцессные события — `SharedFlow`/`StateFlow`, `EventBus`, или просто колбэки.
- Inter-process — обычный `BroadcastReceiver` с permission.

### Безопасность

- **`exported="false"`** — receiver виден только своему приложению (по умолчанию false с targetSdk 31).
- **Permission на broadcast**: `sendBroadcast(intent, "com.x.PERM")` + receiver требует тот же permission.
- **Implicit broadcast от своего приложения** → может быть перехвачен. Используй explicit или `setPackage(packageName)`.

### Когда не нужен Receiver

- Network connectivity — `ConnectivityManager.NetworkCallback` (с Android 7+ broadcast депрекейтнут для CONNECTIVITY_CHANGE).
- Battery — `BatteryManager` query напрямую.
- Time/date — обычно через `Calendar`/`Instant`.

## Подводные камни

- Не отписался динамический receiver в `onStop` → утечка + IllegalStateException при повторном register.
- `goAsync` без `finish()` → ANR.
- Manifest receiver на implicit broadcast в Android 8+ → не вызовется.
- С Android 13+ без `RECEIVER_EXPORTED` флага — `SecurityException` для незащищённых broadcast'ов.
- `onReceive` запускается на main thread → нельзя блокирующую работу.

## Связанные темы

- [[q-06-services]]
- [[q-08-workmanager]]
- [[../11-Security/q-component-export]]

## Follow-up

- Почему с Android 8.0 запретили implicit broadcast в manifest?
- Что делает `goAsync()` и какие у него ограничения?
- Чем заменить LocalBroadcastManager?
