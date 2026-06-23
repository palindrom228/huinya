---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, cancellation, cooperative]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Cancellation корутин: cooperative, CancellationException

## Краткий ответ (TL;DR)

Cancellation в Kotlin — **cooperative**: `cancel()` ставит флаг, но корутина должна сама проверять `isActive` или вызывать suspending функции, которые проверяют. CPU-bound loop без suspension points → не отменяется. Все suspending функции из `kotlinx.coroutines` (delay, yield, await, withContext) — cancellation-aware. Внутри корутины cancel → `CancellationException` бросается на ближайшем suspension point. Ловить — только для cleanup, **обязательно re-throw**.

## Развёрнутый ответ

### Cooperative cancellation

```kotlin
val job = launch(Dispatchers.Default) {
    var i = 0
    while (i < 100_000) {
        // CPU-bound, нет suspension points
        i++
    }
    println("done $i")
}
delay(100)
job.cancel()
job.join()
println("after")  // "done 100000" → "after"
```

Корутина не отменилась — нет точек проверки.

Исправление:
```kotlin
val job = launch(Dispatchers.Default) {
    var i = 0
    while (i < 100_000) {
        ensureActive()    // throws CancellationException if cancelled
        // или: if (!isActive) return@launch
        // или: yield()  — suspending + проверка
        i++
    }
}
```

### Способы кооперации

```kotlin
isActive            // boolean — проверка вручную
ensureActive()      // throws CancellationException
yield()             // suspending + checkpoint + opportunity для других
delay(0)            // тоже работает как checkpoint, но не идиоматично
```

### Все suspending функции — cancellation points

```kotlin
launch {
    delay(1000)        // ← cancel здесь сработает
    api.fetch()        // ← если внутри есть suspension — сработает
    val x = withContext(IO) { ... }  // ← сработает
}
```

Поэтому в типичном Android коде с suspend API всё OK без явных yield.

### try/finally для cleanup

```kotlin
launch {
    val resource = openResource()
    try {
        useResource(resource)
    } finally {
        resource.close()  // выполнится даже при cancel
    }
}
```

### Suspending в finally

```kotlin
launch {
    try {
        work()
    } finally {
        // ❌ delay/withContext здесь бросит CancellationException
        delay(1000)
    }
}
```

Если корутина уже отменена — suspend в finally сразу бросает. Решение:

```kotlin
launch {
    try {
        work()
    } finally {
        withContext(NonCancellable) {
            cleanupSuspending()
        }
    }
}
```

### CancellationException — особый

```kotlin
try {
    longRunningOp()
} catch (e: Exception) {     // ❌ ловит и CancellationException
    log(e)
}
```

Это **проглатывает cancel**. Правильно:

```kotlin
try {
    longRunningOp()
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    log(e)
}
```

Или паттерн `runCatchingCancellable` (см. [[../04-Architecture/q-15-error-handling]]).

### withTimeout / withTimeoutOrNull

```kotlin
try {
    val result = withTimeout(5_000) { api.fetch() }
} catch (e: TimeoutCancellationException) {
    // подтип CancellationException
}

val result = withTimeoutOrNull(5_000) { api.fetch() }  // null при timeout
```

`TimeoutCancellationException` — подтип `CancellationException`, требует особой обработки.

### Cancel ≠ instant

```kotlin
val job = launch { delay(10_000) }
job.cancel()         // вернётся сразу, корутина "Cancelling"
job.join()           // дождётся "Cancelled"
// или:
job.cancelAndJoin()
```

В состоянии Cancelling корутина продолжает выполняться, пока не завершит cleanup.

### Cancel с причиной

```kotlin
job.cancel(CancellationException("User left screen"))
```

Полезно для логирования. Стандартные cancel'ы scope.cancel() не передают reason.

### Отмена snapshot Flow

```kotlin
val job = flow {
    while (true) {
        emit(generate())
        delay(100)
    }
}.launchIn(scope)

job.cancel()   // прервёт delay → выйдет из flow
```

### Отмена async / Deferred

```kotlin
val d = scope.async { heavy() }
d.cancel()
try {
    d.await()
} catch (e: CancellationException) { /* ... */ }
```

`await()` на cancelled Deferred бросает.

### Cancellation через AndroidX

- `Activity.onDestroy()` → `lifecycleScope.cancel()` — отменяет все корутины.
- `ViewModel.onCleared()` → `viewModelScope.cancel()`.
- Composable leaves composition → `rememberCoroutineScope().cancel()`, `LaunchedEffect` отменён.

### CPU-bound code patterns

```kotlin
// Длинный цикл с регулярными checkpoint'ами
List(1_000_000) { it }.forEachIndexed { i, x ->
    if (i % 1000 == 0) ensureActive()
    process(x)
}

// Или yield() для cooperation + giving CPU to others
repeat(1_000_000) { i ->
    if (i % 1000 == 0) yield()
    process(i)
}
```

### Cancellation в callback API

```kotlin
suspend fun fetch() = suspendCancellableCoroutine<User> { cont ->
    val call = api.fetchUser(callback)
    cont.invokeOnCancellation { call.cancel() }  // освобождаем ресурс
}
```

`invokeOnCancellation` — критично для предотвращения leaks (открытые connections, listeners).

## Подводные камни

- **CPU-bound loop без yield** — не отменяется.
- **catch (Exception) без re-throw CancellationException** — проглатывает cancel.
- **`runCatching` в suspend** — ловит CancellationException, корутина не отменяется.
- **`delay/withContext` в finally** — бросает CancellationException, cleanup не завершается. Использовать `NonCancellable`.
- **`job.cancel()` + сразу проверка состояния** — может быть всё ещё Cancelling.
- **callback API без `invokeOnCancellation`** — после cancel callback может ещё прийти, resume на cancelled cont = бесполезно (но не падает — игнорируется).

## Связанные темы

- [[q-04-structured-concurrency]]
- [[q-05-supervisor-job]]
- [[../04-Architecture/q-15-error-handling]]

## Follow-up

- Что значит "cooperative cancellation"?
- Почему `catch (Exception)` опасен в корутинах?
- Зачем `invokeOnCancellation` в `suspendCancellableCoroutine`?
