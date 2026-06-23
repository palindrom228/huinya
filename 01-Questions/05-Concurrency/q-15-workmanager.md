---
type: question
category: concurrency
difficulty: senior
tags: [workmanager, background, jetpack]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# WorkManager: deferrable background work, constraints, coroutines

## Краткий ответ (TL;DR)

`WorkManager` — Jetpack API для **deferrable, guaranteed** background работы (переживает kill, reboot). Использует JobScheduler (API 23+) / AlarmManager fallback. Виды: **OneTimeWorkRequest** / **PeriodicWorkRequest** (мин 15 мин). Constraints (network, charging, idle). `CoroutineWorker.doWork()` — suspend. Цепочки через `beginWith().then()`. Не для немедленной работы — `Service` / `ForegroundService` / прямые корутины.

## Развёрнутый ответ

### Базовый CoroutineWorker

```kotlin
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val url = inputData.getString("url") ?: return Result.failure()
        return try {
            uploader.upload(url)
            Result.success()
        } catch (e: IOException) {
            Result.retry()
        }
    }
}
```

`Result`:
- `success()` — успех, не повторять.
- `failure()` — фатально, не повторять.
- `retry()` — повторить с backoff.

### Enqueue

```kotlin
val request = OneTimeWorkRequestBuilder<UploadWorker>()
    .setInputData(workDataOf("url" to "https://..."))
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)
            .setRequiresCharging(false)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context).enqueue(request)
```

### Periodic

```kotlin
val periodic = PeriodicWorkRequestBuilder<SyncWorker>(
    repeatInterval = 6, repeatIntervalTimeUnit = TimeUnit.HOURS
).build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "daily_sync",
    ExistingPeriodicWorkPolicy.KEEP,   // или UPDATE
    periodic
)
```

Минимальный период — **15 минут** (battery optimization).

### Constraints

```kotlin
Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)   // UNMETERED, METERED, NOT_REQUIRED
    .setRequiresCharging(true)
    .setRequiresBatteryNotLow(true)
    .setRequiresDeviceIdle(true)
    .setRequiresStorageNotLow(true)
    .build()
```

Worker запускается только когда все constraints выполнены.

### Цепочки

```kotlin
WorkManager.getInstance(context)
    .beginWith(downloadWork)
    .then(processWork)
    .then(uploadWork)
    .enqueue()
```

Sequential. Параллельно — список в `.then(listOf(a, b))`.

### Observing state

```kotlin
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(request.id)
    .observe(this) { info ->
        when (info.state) {
            WorkInfo.State.ENQUEUED -> ...
            WorkInfo.State.RUNNING -> ...
            WorkInfo.State.SUCCEEDED -> {
                val output = info.outputData.getString("result")
            }
            WorkInfo.State.FAILED -> ...
            WorkInfo.State.CANCELLED -> ...
        }
    }
```

В Compose: `.getWorkInfoByIdFlow(id).collectAsStateWithLifecycle()`.

### Hilt + Worker

```kotlin
@HiltWorker
class UploadWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repo: UploadRepository
) : CoroutineWorker(context, params) { ... }

@HiltAndroidApp
class App : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory
    override val workManagerConfiguration: Configuration =
        Configuration.Builder().setWorkerFactory(workerFactory).build()
}
```

### Foreground service для долгих задач

```kotlin
class LongWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        setForeground(createForegroundInfo())
        download()
        return Result.success()
    }

    private fun createForegroundInfo() = ForegroundInfo(
        NOTIFICATION_ID,
        NotificationCompat.Builder(applicationContext, CHANNEL_ID)
            .setContentTitle("Downloading")
            .setSmallIcon(R.drawable.ic_download)
            .build()
    )
}
```

`setForeground()` обязательно для работы дольше ~10 минут на новых Android.

### Expedited work (API 31+)

```kotlin
OneTimeWorkRequestBuilder<X>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
```

Запускается почти сразу, для важных коротких задач. Лимит quota от системы.

### Unique work

```kotlin
WorkManager.getInstance(context).enqueueUniqueWork(
    "upload_user_data",
    ExistingWorkPolicy.REPLACE,   // KEEP, APPEND, APPEND_OR_REPLACE
    request
)
```

Гарантия: только одна work с этим именем в очереди.

### Cancel

```kotlin
WorkManager.getInstance(context).cancelWorkById(request.id)
WorkManager.getInstance(context).cancelUniqueWork("daily_sync")
WorkManager.getInstance(context).cancelAllWorkByTag("sync")
```

### Когда использовать

✅ Sync данных (опционально по сети, можно отложить).
✅ Логи / analytics батчи.
✅ Periodic backup.
✅ Загрузка / скачивание файлов.
✅ Cleanup БД.

❌ **Не для немедленной** работы — используй прямые корутины или Service.
❌ **Не для UI-bound** работы — VM scope.
❌ **Не для real-time** — есть delay enforcement.

### WorkManager vs альтернативы

- **Coroutines (viewModelScope)** — для UI-связанной асинхронной работы, не переживает kill.
- **Service / ForegroundService** — long-running, пока app жив.
- **AlarmManager** — точное время, но без guarantee work-completion.
- **JobScheduler** — низкоуровневый (WorkManager его использует).
- **WorkManager** — guaranteed, deferrable, с constraints.

### Тестирование

```kotlin
@RunWith(AndroidJUnit4::class)
class UploadWorkerTest {
    @Before fun setup() {
        WorkManagerTestInitHelper.initializeTestWorkManager(context)
    }

    @Test fun success() {
        val request = OneTimeWorkRequestBuilder<UploadWorker>().build()
        val wm = WorkManager.getInstance(context)
        wm.enqueue(request).result.get()
        val info = wm.getWorkInfoById(request.id).get()
        assertEquals(WorkInfo.State.SUCCEEDED, info.state)
    }
}
```

`TestDriver` для управления constraints в тестах:

```kotlin
WorkManagerTestInitHelper.getTestDriver(context)?.setAllConstraintsMet(request.id)
```

### Battery / Doze mode

- Periodic минимум 15 мин — нельзя обойти.
- Doze mode откладывает не-expedited work.
- App standby — частота снижается для редко используемых apps.

### Migration с GcmTaskService / Firebase JobDispatcher

Полностью устарели. WorkManager — единственный современный API.

## Подводные камни

- `PeriodicWorkRequest` < 15 минут → unmet constraint, никогда не выполнится.
- `enqueue` без `enqueueUniqueWork` при повторных вызовах → размножение work.
- Worker без Hilt factory → `IllegalStateException: Could not instantiate`.
- `Result.failure()` вместо `retry()` для transient ошибок → потеря работы.
- Большой `inputData` (>10 KB) → IllegalStateException, лимит Data bundle.
- `setForeground` без notification channel (API 26+) → crash.
- Возврат из `doWork()` после `withContext(NonCancellable)` для cleanup — задержка cancellation.

## Связанные темы

- [[../02-Android-Core/q-services-foreground-bound]]
- [[../02-Android-Core/q-notifications]]
- [[q-06-cancellation]]

## Follow-up

- Когда использовать WorkManager, а когда прямые корутины?
- Что такое constraints и зачем?
- Чем `Result.retry()` отличается от `failure()`?
