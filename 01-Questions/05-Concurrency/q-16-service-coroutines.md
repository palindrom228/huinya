---
type: question
category: concurrency
difficulty: senior
tags: [service, foreground, lifecyclescope]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Service + корутины: lifecycle, ForegroundService, scope

## Краткий ответ (TL;DR)

`Service` — компонент без UI для долгих операций. Создай scope: `CoroutineScope(SupervisorJob() + Dispatchers.IO)` и **отмени в `onDestroy`**. Альтернатива — `lifecycle-service` artifact даёт `LifecycleService` + `lifecycleScope`. **ForegroundService** обязателен для долгих задач (показывает notification, не убивается агрессивно). С Android 14 + `foregroundServiceType` обязателен. Для отложенной работы — WorkManager, не Service.

## Развёрнутый ответ

### Service со scope

```kotlin
class UploadService : Service() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        scope.launch {
            uploadFiles()
            stopSelf(startId)
        }
        return START_STICKY
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

### LifecycleService (рекомендуется)

```kotlin
class UploadService : LifecycleService() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        super.onStartCommand(intent, flags, startId)
        lifecycleScope.launch {
            uploadFiles()
            stopSelf(startId)
        }
        return START_STICKY
    }
}
```

`lifecycleScope` автоматически отменяется в `onDestroy`. Нужна зависимость `androidx.lifecycle:lifecycle-service`.

### ForegroundService

```kotlin
class DownloadService : LifecycleService() {
    override fun onCreate() {
        super.onCreate()
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Downloading")
            .setSmallIcon(R.drawable.ic_download)
            .build()

        startForeground(NOTIFICATION_ID, notification)
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        lifecycleScope.launch {
            download()
            stopSelf(startId)
        }
        return START_STICKY
    }
}
```

`startForeground` **в течение 5 секунд** после `startForegroundService()` — иначе ANR.

### Android 14+ foregroundServiceType

Manifest:

```xml
<service
    android:name=".DownloadService"
    android:foregroundServiceType="dataSync" />
```

Типы: `camera`, `location`, `mediaPlayback`, `mediaProjection`, `microphone`, `phoneCall`, `dataSync`, `health`, `shortService`, `specialUse`, `systemExempted`.

В коде:
```kotlin
ServiceCompat.startForeground(
    this,
    NOTIFICATION_ID,
    notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
)
```

### START_STICKY vs START_NOT_STICKY vs START_REDELIVER_INTENT

- **START_STICKY** — система пересоздаст Service после kill, intent = null. Для долгих background задач.
- **START_NOT_STICKY** — не пересоздаст. Для one-shot.
- **START_REDELIVER_INTENT** — пересоздаст и доставит последний intent. Для важных задач (загрузка с сохранением intent).

### Bound Service

```kotlin
class PlayerService : LifecycleService() {
    private val binder = PlayerBinder()

    inner class PlayerBinder : Binder() {
        fun getService(): PlayerService = this@PlayerService
    }

    override fun onBind(intent: Intent): IBinder {
        super.onBind(intent)
        return binder
    }
}

// Activity:
private val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        playerService = (service as PlayerBinder).getService()
    }
    override fun onServiceDisconnected(name: ComponentName) { playerService = null }
}

bindService(Intent(this, PlayerService::class.java), connection, BIND_AUTO_CREATE)
```

### Sharing data со Service через Flow

```kotlin
class DownloadService : LifecycleService() {
    companion object {
        private val _progress = MutableStateFlow(0)
        val progress: StateFlow<Int> = _progress.asStateFlow()
    }

    override fun onStartCommand(...): Int {
        lifecycleScope.launch {
            download { current -> _progress.value = current }
        }
        return START_STICKY
    }
}

// VM:
class DownloadVm : ViewModel() {
    val progress = DownloadService.progress
}
```

Companion StateFlow — простой способ. Чище — bound service / repository.

### Когда Service vs WorkManager vs корутины

| | Service | WorkManager | Coroutines |
|---|---------|-------------|------------|
| Live UI tied | ❌ | ❌ | ✅ |
| Survives app death | ✅ (foreground) | ✅ | ❌ |
| Survives reboot | ❌ | ✅ | ❌ |
| Deferrable | ❌ | ✅ | ❌ |
| Examples | music player, navigation | sync, backup | API call from VM |

### Foreground service restrictions (Android 12+)

- Нельзя стартовать из background (с некоторыми исключениями).
- Whitelist: notification action, geofence, system broadcast.
- Иначе `ForegroundServiceStartNotAllowedException`.

Решение: использовать **expedited WorkManager** или запускать Service из активного UI.

### Stopping

```kotlin
// Из самого Service:
stopSelf()       // полная остановка
stopSelf(startId) // только если последний startId

// Снаружи:
context.stopService(Intent(context, MyService::class.java))
```

### Pitfall: long work in onStartCommand main thread

```kotlin
// ❌ блокирует Main, ANR
override fun onStartCommand(...): Int {
    Thread.sleep(5000)  // или sync I/O
    return START_STICKY
}
```

Service по умолчанию работает на **main thread**. Всегда корутина / Thread.

### Pitfall: scope не отменён

```kotlin
class BadService : Service() {
    private val scope = CoroutineScope(Dispatchers.IO)
    // забыли scope.cancel() в onDestroy → утечка
}
```

### IntentService (deprecated)

Старый сериализованный Service. Замена — `JobIntentService` (тоже deprecated) → **WorkManager**.

### Тестирование

```kotlin
@Test fun serviceUploads() {
    val intent = Intent(context, UploadService::class.java)
    context.startService(intent)
    // ServiceTestRule для контроля lifecycle
}
```

`androidx.test:rules` → `ServiceTestRule`.

## Подводные камни

- Service на Main thread + блокировка → ANR.
- Забыл `scope.cancel()` в `onDestroy` → корутины-зомби.
- `startForeground` позже 5 сек после `startForegroundService` → ANR + RemoteServiceException.
- Android 14+ без `foregroundServiceType` → SecurityException.
- `START_STICKY` без проверки на `intent == null` → NPE при пересоздании.
- Использование Service для отложенной работы → правильнее WorkManager.
- Background start Service на Android 12+ → ForegroundServiceStartNotAllowedException.

## Связанные темы

- [[../02-Android-Core/q-services-foreground-bound]]
- [[q-15-workmanager]]
- [[q-02-coroutine-scope-context]]

## Follow-up

- Какой scope использовать в Service?
- В чём разница Service vs ForegroundService vs WorkManager?
- Зачем `foregroundServiceType` с Android 14?
