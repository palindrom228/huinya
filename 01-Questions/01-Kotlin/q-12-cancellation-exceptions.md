---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines, exceptions]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Отмена и обработка исключений в корутинах: `CancellationException`, `SupervisorJob`, `CoroutineExceptionHandler`

## Краткий ответ (TL;DR)

Отмена корутины — **кооперативная**: бросается `CancellationException`, suspend-точки её проверяют. **`CancellationException` не обрабатывается как ошибка** — родитель не отменяется. Все другие исключения «всплывают вверх» по дереву Job → отменяют родителя и братьев, если не `SupervisorJob`. `CoroutineExceptionHandler` ловит только **uncaught** исключения на корневых корутинах `launch` (для `async` исключение приходит в `await`).

## Развёрнутый ответ

### Кооперативная отмена

```kotlin
val job = launch {
    repeat(1000) { i ->
        delay(100)            // suspend — проверяет cancel
        println("tick $i")
    }
}
delay(500)
job.cancel()                  // job в состоянии Cancelling
job.join()                    // ждём пока ребёнок завершится
```

Если внутри корутины **тяжёлый CPU-loop без suspend** — он не отменится:

```kotlin
launch {
    while (true) { compute() }   // не реагирует на cancel
}
```

Решение: периодически вызывать `yield()` или `ensureActive()`, либо проверять `isActive`.

### CancellationException

`CancellationException` — особый класс. Coroutine machinery бросает её внутрь корутины, чтобы остановить, но **не считает за ошибку**:

```kotlin
try {
    delay(1000)
} catch (e: Exception) {        // ← поймает CancellationException
    // плохо: проглотили отмену
}
```

**Правило:** `catch (e: CancellationException) { throw e }` или вообще не ловить — пусть пробрасывается. Альтернатива: `catch (e: Exception)` → `if (e is CancellationException) throw e else handle(e)`.

С Kotlin 1.8+ есть удобный паттерн через `runCatching`:

```kotlin
val result = runCatching { riskyCall() }
    .onFailure { if (it is CancellationException) throw it }
```

### withContext / NonCancellable

Cleanup, который **должен** выполниться даже при отмене:

```kotlin
try {
    work()
} finally {
    withContext(NonCancellable) {
        releaseResource()      // не отменится, даже если родитель в Cancelling
    }
}
```

### Распространение исключений

```kotlin
coroutineScope {           // НЕ supervisor
    launch { throw IOException() }
    launch { /* отменится */ }
}                          // coroutineScope перебросит IOException
```

```kotlin
supervisorScope {          // дети независимы
    launch { throw IOException() }   // упала только эта
    launch { /* живёт */ }
}
```

### CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, e -> log(e) }
val scope = CoroutineScope(SupervisorJob() + handler)
scope.launch { throw RuntimeException("x") }   // → handler
```

Ловит только **uncaught на root coroutine**. Для `async` — исключение хранится в `Deferred` и выбрасывается на `await()`.

### Try-catch для корутин

```kotlin
try {
    coroutineScope {
        launch { throw IOException() }   // упадёт всё; try поймает после scope
    }
} catch (e: IOException) { ... }
```

В то время как:

```kotlin
launch {
    try { riskyApi() } catch (e: Exception) { handle(e) }  // ✅ обычный try
}
```

## Подводные камни

- `e: Exception` ловит `CancellationException` → утечка отмены.
- `try { } catch (e: Throwable) { }` вокруг тела корутины — может «съесть» отмену → корутина перестанет реагировать.
- `async { throw }` без `await` → исключение пропадёт; в `supervisorScope` — точно.
- `runBlocking { }` пробрасывает исключения в caller.
- ExceptionHandler не работает в нестроковых корутинах (в детях): handler берётся с **root coroutine**.

## Связанные темы

- [[q-10-coroutines-basics]]
- [[q-11-coroutine-context]]

## Follow-up

- Что произойдёт, если в `finally` запустить новую корутину после cancel?
- Как корректно отменить CPU-bound цикл?
- Почему `async` хранит исключение, а `launch` бросает сразу?
