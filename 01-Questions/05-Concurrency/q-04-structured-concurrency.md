---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, structured-concurrency, exceptions]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Structured concurrency: parent-child, exceptions

## Краткий ответ (TL;DR)

**Structured concurrency** = иерархия: каждая корутина имеет parent, parent не завершается пока не завершены все children. Исключение в child → cancel parent → cancel всех siblings (если не `SupervisorJob`). Cancellation идёт **сверху вниз** (parent→children), exception propagation — **снизу вверх** (child→parent). `coroutineScope { }` — суспенд функция, ждёт всех children, fail-fast. `supervisorScope` — children независимы.

## Развёрнутый ответ

### Принципы

1. **Lifetime**: parent не завершён, пока все children не завершены.
2. **Cancellation**: cancel(parent) → cancel(всех children) рекурсивно.
3. **Exceptions**: unhandled exception в child → cancel(parent) + propagate up.
4. **No leaks**: scope.cancel() гарантирует, что никакая корутина не "сбежит".

### Иерархия

```
viewModelScope.launch {        // parent1
    launch { delay(1000) }     // child1.1
    launch { delay(2000) }     // child1.2
    val data = async { fetch() } // child1.3
    data.await()
}  // parent1 завершится после всех children
```

### coroutineScope

```kotlin
suspend fun loadAll(): Pair<A, B> = coroutineScope {
    val a = async { loadA() }
    val b = async { loadB() }
    a.await() to b.await()
}
```

- Suspend функция (не extension на CoroutineScope!).
- Создаёт sub-scope, ждёт всех children.
- Любой fail → cancel siblings + throw наружу.

### supervisorScope

```kotlin
suspend fun loadIndependent() = supervisorScope {
    launch { riskyOp1() }     // упал — НЕ отменит риски2
    launch { riskyOp2() }
}
```

Дети **независимы** — fail одного не отменяет других. Полезно для параллельных задач, где частичные результаты OK.

### Exception propagation

```kotlin
val scope = CoroutineScope(Job())
scope.launch {                        // parent
    launch { throw RuntimeException() }  // child — exception
    launch { delay(1000); println("never prints") }  // sibling — отменён
}
// scope тоже отменён → весь scope нерабочий
```

С `SupervisorJob`:

```kotlin
val scope = CoroutineScope(SupervisorJob())
scope.launch { throw RuntimeException() }  // только этот падает
scope.launch { delay(1000); println("OK") } // нормально работает
```

### launch vs async — exceptions

```kotlin
// launch — exception throws сразу через CoroutineExceptionHandler
scope.launch { throw RuntimeException() }  // handler сработает

// async — exception хранится в Deferred, throws только на await()
val d = scope.async { throw RuntimeException() }  // ничего не происходит
d.await()  // здесь throws
```

**НО** в обычном (не Supervisor) scope async exception всё равно отменяет parent. Только в supervisorScope async тихо хранит exception до await.

### Пример: загрузка списка изображений

```kotlin
// ❌ один fail отменяет всё
suspend fun loadAll(urls: List<String>): List<Bitmap> = coroutineScope {
    urls.map { url -> async { loadImage(url) } }.awaitAll()
}

// ✅ частичные результаты
suspend fun loadAllBestEffort(urls: List<String>): List<Bitmap?> = supervisorScope {
    urls.map { url ->
        async {
            try { loadImage(url) } catch (e: Exception) { null }
        }
    }.awaitAll()
}
```

### CancellationException

```kotlin
try {
    childJob.await()
} catch (e: CancellationException) {
    throw e   // ВСЕГДА re-throw
} catch (e: Exception) {
    // handle other
}
```

`CancellationException` — это сигнал для structured concurrency, не баг. Ловить только для cleanup, **всегда re-throw**.

### Coroutine builders сводка

| Builder | Возвращает | Exception behavior |
|---------|-----------|---------------------|
| `launch` | Job | бросает в handler сразу |
| `async` | Deferred | хранит до await() |
| `runBlocking` | T | бросает синхронно |
| `coroutineScope` | T | suspend, ждёт всех, fail-fast |
| `supervisorScope` | T | suspend, children независимы |
| `withContext` | T | suspend, меняет context |

### Best practice cancellation

```kotlin
suspend fun doWork() {
    try {
        riskyCall()
    } catch (c: CancellationException) {
        // cleanup if needed
        throw c   // обязательно
    } catch (e: IOException) {
        // handle business error
    }
}
```

`runCatching` ловит `CancellationException` — **опасно** (см. [[../04-Architecture/q-15-error-handling]]).

### Job state machine

```
NEW → ACTIVE → COMPLETING → COMPLETED
              ↓
              CANCELLING → CANCELLED
```

`isActive`, `isCancelled`, `isCompleted` — три флага.

### NonCancellable

```kotlin
withContext(NonCancellable) {
    // даже при cancel — этот блок завершится
    cleanupResources()
}
```

Использовать **только** для гарантированного cleanup в `finally`. Никогда для основной логики.

```kotlin
try {
    work()
} finally {
    withContext(NonCancellable) {
        releaseLock()
    }
}
```

## Подводные камни

- **Ловишь `Exception` и не re-throw** → проглатываешь `CancellationException`, scope не отменяется.
- **`async` без `await` в обычном scope** — exception всё равно бросится через parent. Только `supervisorScope`/`SupervisorJob` тихий.
- **`GlobalScope.launch`** — нет structured concurrency, корутина "сбегает".
- **`Job()` в child context** — отрывает от родителя, breaks cancellation.
- **`withContext(NonCancellable) { work() }`** — work не отменится, повисает навсегда при cancel.
- **fork & forget**: `scope.launch { ... }` без отслеживания → нет гарантии завершения при scope.cancel() (на самом деле есть, но люди забывают).

## Связанные темы

- [[q-02-coroutine-scope-context]]
- [[q-05-supervisor-job]]
- [[q-06-cancellation]]
- [[q-08-exception-handler]]

## Follow-up

- Что такое structured concurrency и зачем?
- Чем `coroutineScope` отличается от `supervisorScope`?
- Почему `CancellationException` нужно re-throw?
