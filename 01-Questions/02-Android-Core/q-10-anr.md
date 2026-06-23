---
type: question
category: android-core
difficulty: senior
tags: [anr, performance, main-thread]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 1
times_passed: 0
status: learning
---

# ANR: когда возникает, как диагностировать, как избежать

## Краткий ответ (TL;DR)

**ANR** (Application Not Responding) — система убеждена, что приложение не отвечает. Срабатывает при: Main thread заблокирован > 5 сек на input event, BroadcastReceiver > 10 сек, Foreground service не вызвал `startForeground` за 5 сек, Content Provider > timeout. Диагностика — `/data/anr/traces.txt`, Play Console ANR rate, Perfetto/Systrace. Лекарство — никогда не блокировать main thread.

## Развёрнутый ответ

### Триггеры

| Где | Таймаут | Что |
|-----|---------|-----|
| Input event (touch, key) | 5 сек | пользователь tap'нул и не получил ответа |
| BroadcastReceiver.onReceive | 10 сек (FG), 60 сек (BG) | долгая работа в receiver |
| Service.onStartCommand / onBind | 20 сек (FG), 200 сек (BG) | блок на main |
| Foreground service startup | 5 сек | не вызвал startForeground |
| ContentProvider publish | 10 сек | долгий onCreate |
| JobScheduler | ~10 мин на onStartJob возврат | редкий |

### Что НЕ ANR (но похоже)

- **Frozen frames** (> 700ms на кадр) — фиксируются в Play Console, но не ANR.
- **App startup** — медленный, но если main не заблокирован — не ANR.

### Главные причины

1. **Disk IO на main**: чтение файла, SharedPreferences `commit()` (не `apply`).
2. **DB queries** на main (Room без suspend/Flow).
3. **Сетевые запросы** на main (давно запрещено в strict mode, но через legacy код встречается).
4. **Bitmap decoding** больших картинок на main.
5. **JSON parse** больших ответов на main (если без библиотек).
6. **Inflate тяжёлого layout**.
7. **Deadlock** между main thread и worker.
8. **Lock contention** — `synchronized` ждёт долгого worker thread.
9. **Binder transaction** — IPC к слабоотвечающему system_server / другому процессу.
10. **WebView** load на main блокирует UI thread иногда.

### Диагностика

#### 1. ANR trace

При ANR система собирает `traces.txt` со стеками всех потоков всех процессов в `/data/anr/`. На Play Console это видно в **Android vitals**.

Анализ:
- Найти стек main thread.
- Если на `nativePollOnce` → main ждёт работу — ANR вызван не main thread'ом, а внешним (system_server занят? IPC?).
- Если на `BinderProxy.transact` → IPC долго возвращается.
- Если на `SQLiteConnection.nativeExecute` → DB на main.

#### 2. StrictMode

```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(StrictMode.ThreadPolicy.Builder()
        .detectAll().penaltyLog().build())
    StrictMode.setVmPolicy(StrictMode.VmPolicy.Builder()
        .detectLeakedClosableObjects()
        .detectActivityLeaks()
        .build())
}
```

Ловит disk/network на main, leaks, untagged sockets — в дебаге.

#### 3. ApplicationExitInfo (Android 11+)

```kotlin
val am = getSystemService<ActivityManager>()!!
val exits = am.getHistoricalProcessExitReasons(packageName, 0, 10)
exits.filter { it.reason == ApplicationExitInfo.REASON_ANR }
     .forEach { Log.w("ANR", it.description) }
```

С трейсом — можно отправить себе для аналитики.

#### 4. Perfetto / Systrace

Системные трейсы — видно, что main thread делал в момент freeze.

#### 5. Production monitoring

- Firebase Crashlytics (ANR с Android 11+).
- Sentry, Bugsnag.
- Custom — `Choreographer.FrameCallback` для frozen frames.

### Решения

- Любая блокирующая работа — в Coroutine с `Dispatchers.IO`/`Default`.
- Room — `suspend` DAO или `Flow`.
- SharedPreferences `apply()`, не `commit()`. Лучше — DataStore.
- Bitmap — `inSampleSize`, off-main decode (Coil/Glide).
- WebView — `setLayerType(HARDWARE)` для тяжёлых страниц.
- Избегать deadlock'ов: единая дисциплина блокировок, или вообще без них (Mutex + suspend).

## Подводные камни

- `runBlocking` в обработчике клика → 100% ANR при медленной операции.
- `Thread.sleep` где угодно на main.
- Synchronous Firebase Performance trace start — раньше блокировал init.
- BroadcastReceiver, который синхронно делает работу > 10 сек — ANR.
- `LiveData.value` запись с background thread → exception, но `postValue` без проверки — coalesce, теряет промежуточные.

## Связанные темы

- [[../07-Performance-Memory/q-strictmode]]
- [[../07-Performance-Memory/q-startup]]
- [[q-06-services]]

## Follow-up

- Как отличить ANR от просто медленной анимации?
- Что делать, если ANR-trace показывает `nativePollOnce` на main thread?
- Чем `apply()` отличается от `commit()` в SharedPreferences?
