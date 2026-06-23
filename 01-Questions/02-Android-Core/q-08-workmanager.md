---
type: question
category: android-core
difficulty: senior
tags: [workmanager, background]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# WorkManager: когда использовать, OneTime vs Periodic, constraints, expedited, chains

## Краткий ответ (TL;DR)

`WorkManager` — стандарт для **отложенной/гарантированной** фоновой работы. Переживает kill процесса, restart девайса, batched по constraints (Doze, network). НЕ для немедленной работы и НЕ для UI-зависимых задач. Поддерживает chains, unique work, expedited (с приоритетом FGS-like), Coroutines/Rx.

## Развёрнутый ответ

### Когда WorkManager

| Нужно | WorkManager? |
|-------|--------------|
| Сохранить лог через 5 мин | ✅ |
| Синхронизировать данные раз в день | ✅ Periodic |
| Загрузить файл в фоне | ✅ |
| Отправить crash report | ✅ |
| Немедленная работа сейчас | ❌ — используй корутины в scope |
| Foreground UI работа | ❌ |
| Точно в 14:00 | ❌ — `AlarmManager.setExactAndAllowWhileIdle` |
| Push handling | FCM + WM для отложки ок |

### OneTime vs Periodic

```kotlin
val work = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.UNMETERED)
        .setRequiresCharging(true)
        .build())
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .setInputData(workDataOf("id" to 42L))
    .build()

WorkManager.getInstance(context).enqueue(work)
```

**Periodic** — минимум **15 минут** интервал. Не гарантирует точное время (batch).

```kotlin
PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
```

### Constraints

- `setRequiredNetworkType(CONNECTED | UNMETERED | METERED | NOT_REQUIRED)`
- `setRequiresCharging(true)`
- `setRequiresBatteryNotLow(true)`
- `setRequiresStorageNotLow(true)`
- `setRequiresDeviceIdle(true)`
- `addContentUriTrigger(uri, triggerForDescendants)` — при изменении URI

### Worker / CoroutineWorker

```kotlin
class UploadWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        try {
            api.upload(inputData.getString("path")!!)
            Result.success()
        } catch (e: IOException) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}
```

`Result.retry()` → перепланируется с backoff. `Result.failure()` → перестаёт пытаться.

### Expedited work

С Android 12+ можно пометить срочной — выполняется почти сразу, с ограничениями системы (до 10 мин). Под капотом — короткий foreground service.

```kotlin
val work = OneTimeWorkRequestBuilder<W>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
```

Worker должен реализовать `getForegroundInfo()` для уведомления (на API < 31 показывается как FGS).

### Unique work

```kotlin
WorkManager.getInstance(ctx).enqueueUniqueWork(
    "sync", ExistingWorkPolicy.KEEP, work
)
```

Политики: `REPLACE`, `KEEP`, `APPEND`, `APPEND_OR_REPLACE`.

Для periodic: `enqueueUniquePeriodicWork(name, ExistingPeriodicWorkPolicy.UPDATE, periodic)`.

### Chains

```kotlin
WorkManager.getInstance(ctx)
    .beginWith(workA)
    .then(workB)
    .then(workC)
    .enqueue()
```

`outputData` одного worker'а — `inputData` следующего. Параллельные ветки — через `combine()`.

### Под капотом

- Использует `JobScheduler` (API 23+), `AlarmManager` + `BroadcastReceiver` (< 23).
- State в локальной SQLite (`androidx.work.workdb`).
- Переживает reboot, kill процесса.
- Между процессами — один `WorkManager` инстанс (с remote workers для multi-process).

### Тестирование

`androidx.work.testing.WorkManagerTestInitHelper` + `TestDriver` для управления временем/constraints в тестах.

## Подводные камни

- Periodic меньше 15 мин — нельзя (Doze + system batching).
- Expedited имеет квоту в системе (зависит от standby bucket) — не злоупотреблять.
- WorkManager в multi-process приложении — нужен `androidx.work.multiprocess` для remote workers.
- `Hilt` worker injection — нужна `@HiltWorker` + `HiltWorkerFactory` в `WorkManagerInitializer`.
- `Worker.doWork()` (не CoroutineWorker) выполняется на **executor pool**, не main — но блокирующий код там OK; в `CoroutineWorker.doWork` нужно явно `withContext(IO)` для blocking IO.

## Связанные темы

- [[q-06-services]]
- [[q-07-broadcastreceiver]]
- [[q-09-process-death]]

## Follow-up

- Чем expedited work отличается от обычного?
- Как обеспечить, чтобы заданная sync-работа не запускалась дважды?
- Что произойдёт с running worker'ом при kill процесса?
