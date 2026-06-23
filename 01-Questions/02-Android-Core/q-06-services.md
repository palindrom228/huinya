---
type: question
category: android-core
difficulty: senior
tags: [service, foreground, background-limits]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Service: started vs bound vs foreground. Background execution limits (Oreo+, 12+)

## Краткий ответ (TL;DR)

**Started** — `startService`, работает пока сам не остановится. **Bound** — `bindService`, живёт пока есть клиенты. **Foreground** — обязательное notification, не убивается в background, нужен явный тип с Android 14. С Android 8.0 background apps не могут запускать обычные Service (только foreground). С Android 12+ есть запрет на запуск foreground service из background (за редкими исключениями).

## Развёрнутый ответ

### Started Service

```kotlin
val intent = Intent(context, MyService::class.java)
ContextCompat.startForegroundService(context, intent)  // 8.0+
```

- `onStartCommand` вызывается при каждом `startService`.
- Возвращает `START_STICKY` / `START_NOT_STICKY` / `START_REDELIVER_INTENT` — что делать после kill.
- Останавливается через `stopSelf()` или `stopService()`.

### Bound Service

```kotlin
val conn = object : ServiceConnection { ... }
bindService(intent, conn, Context.BIND_AUTO_CREATE)
```

- Возвращает `IBinder` для IPC / прямого вызова.
- Когда все клиенты `unbind` → Service уничтожается.
- Можно совмещать started + bound.

### Foreground Service

Обязателен для долгой работы видимой пользователю (плеер, навигация, запись звонка). С Android 8.0:

```kotlin
override fun onCreate() {
    super.onCreate()
    val notif = NotificationCompat.Builder(this, CH_ID)
        .setSmallIcon(R.drawable.ic)
        .setContentTitle("Playing")
        .build()
    startForeground(NOTIF_ID, notif)
}
```

Должен быть вызван **в течение 5 секунд** после `startForegroundService`, иначе ANR + кill (RemoteServiceException на 12+).

#### Android 14: foregroundServiceType

```xml
<service
    android:name=".PlayerService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```

```kotlin
startForeground(id, notif, ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK)
```

Типы: `mediaPlayback`, `location`, `phoneCall`, `dataSync`, `mediaProjection`, `camera`, `microphone`, `health`, `connectedDevice`, `shortService`, `specialUse`, `systemExempted`. Каждому нужен соответствующий permission.

### Background execution limits

#### Android 8.0 (API 26)

Background приложение **не может** запустить обычный Service. Только:
- `startForegroundService` (с обязательным `startForeground` в 5 сек).
- WorkManager / JobScheduler / AlarmManager.

#### Android 12 (API 31)

**Запрет запуска foreground service из background** (`ForegroundServiceStartNotAllowedException`). Исключения: VPN, FCM high-priority push, alarm, viewable activity (запуск из видимой), bind с FLAG `BIND_ALLOW_OOM_MANAGEMENT` от system, и т.д.

Решение: использовать **expedited WorkManager** для фоновой срочной работы.

#### Android 14

Каждый FGS требует explicit `foregroundServiceType` и соответствующий permission (`FOREGROUND_SERVICE_*`).

### Когда что использовать

| Задача | Решение |
|--------|---------|
| Скачать файл/синхронизация | `WorkManager` (с constraints) |
| Срочная фоновая работа | Expedited WorkManager |
| Музыка / навигация | Foreground Service |
| Долгая работа без UI, видимая пользователю | Foreground Service |
| IPC между процессами | Bound Service + Messenger/AIDL |
| Реакция на push | FCM + локальная обработка / WorkManager |
| Короткая фоновая задача < 10 минут | `WorkManager.OneTime` |

### IntentService → JobIntentService → WorkManager

- `IntentService` deprecated в Android 11.
- `JobIntentService` (compat) тоже устаревает.
- Замена — `WorkManager`.

## Подводные камни

- Не вызвал `startForeground` за 5 сек → `RemoteServiceException` / kill.
- `bindService(BIND_AUTO_CREATE)` без unbind в `onStop` → утечка контекста.
- `START_STICKY` пересоздаст Service с `null` intent — обрабатывай.
- В Android 12+ запуск foreground service из BroadcastReceiver в background — упадёт.
- `stopSelf(startId)` останавливает только если это последний startId — иначе игнорирует.

## Связанные темы

- [[q-07-broadcastreceiver]]
- [[q-08-workmanager]]
- [[q-04-intents]]

## Follow-up

- Что произойдёт, если не вызвать `startForeground` после `startForegroundService`?
- Чем expedited work отличается от обычной WorkManager job?
- Когда `bindService` лучше, чем `startService`?
