---
type: question
category: concurrency
difficulty: middle
tags: [coroutines, builders, async]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# withContext / launch / async — отличия и use cases

## Краткий ответ (TL;DR)

**`launch`** — fire-and-forget, возвращает `Job` (нет результата). **`async`** — параллельная задача с результатом, возвращает `Deferred<T>`, забирается через `await()`. **`withContext`** — suspend, меняет context (обычно dispatcher), ждёт результат, возвращает T. Параллельность: `async + async + await` параллельно; `withContext + withContext` — последовательно. `runBlocking` — мост в blocking мир, **не** в production коде на Main.

## Развёрнутый ответ

### launch

```kotlin
val job: Job = scope.launch {
    repo.refresh()
}
job.cancel()
job.join()
```

- Fire-and-forget или task с управлением через Job.
- Нет возвращаемого значения.
- Exception → propagates до scope (или handler).

### async

```kotlin
val deferred: Deferred<User> = scope.async {
    api.fetchUser(id)
}
val user = deferred.await()
```

- Аналог Future.
- Используй для **параллельных** задач с результатом.
- Exception → хранится в Deferred, бросается на `await()` (но в обычном scope всё равно отменит parent).

### withContext

```kotlin
suspend fun load(): Data = withContext(Dispatchers.IO) {
    api.fetchBlocking()
}
```

- Suspend функция, не builder.
- Меняет context (обычно dispatcher) для блока.
- Возвращает T напрямую.
- Sequential — ждёт завершения.

### Параллельность

```kotlin
// ❌ последовательно (хоть и на одном dispatcher)
suspend fun loadSeq(): Pair<A, B> {
    val a = withContext(IO) { fetchA() }   // 1 сек
    val b = withContext(IO) { fetchB() }   // ещё 1 сек
    return a to b                          // total: 2 сек
}

// ✅ параллельно
suspend fun loadPar(): Pair<A, B> = coroutineScope {
    val a = async(IO) { fetchA() }
    val b = async(IO) { fetchB() }
    a.await() to b.await()                 // total: 1 сек
}
```

### async без coroutineScope

```kotlin
// ❌ нет structured concurrency
fun load() = scope.async { fetch() }  // result хранится до await

// ✅
suspend fun load() = coroutineScope {
    val d = async { fetch() }
    d.await()
}
```

`async` напрямую на scope без обёртки = exception потеряется до await.

### CoroutineStart режимы

```kotlin
scope.launch(start = CoroutineStart.LAZY) { /* ... */ }.start()
```

- `DEFAULT` — запускается сразу.
- `LAZY` — ждёт `start()` / `join()` / `await()`.
- `ATOMIC` — нельзя отменить до первого suspension.
- `UNDISPATCHED` — стартует в текущем потоке без диспатча, до первого suspension.

### runBlocking

```kotlin
fun main() = runBlocking {
    val data = api.fetch()
    println(data)
}
```

- Блокирует поток до завершения.
- Используется в `main()`, тестах (deprecated в пользу `runTest`).
- **Никогда** на Main thread в production (deadlock potential).

### Возвращаемые типы

| Builder | Тип | Дождаться | Cancel |
|---------|-----|-----------|--------|
| `launch` | `Job` | `.join()` | `.cancel()` |
| `async` | `Deferred<T>` | `.await()` | `.cancel()` |
| `withContext` | `T` | автоматически | через cancel scope |
| `runBlocking` | `T` | автоматически | cancel вне |

### withContext оптимизация

```kotlin
withContext(Main.immediate) {
    if (alreadyOnMain) {
        // не диспатчит, выполняет inline
    }
}
```

Также: `withContext(EmptyCoroutineContext)` — no-op оптимизация в свежих версиях.

### Pattern: main-safe repository

```kotlin
class Repo {
    suspend fun load(): Data = withContext(Dispatchers.IO) {
        diskCache.read() ?: api.fetchBlocking().also { diskCache.write(it) }
    }
}

// VM вызывает без обёртки:
viewModelScope.launch {
    val data = repo.load()
    _state.value = data
}
```

### awaitAll / joinAll

```kotlin
suspend fun loadMany(ids: List<Long>): List<User> = coroutineScope {
    ids.map { id -> async { fetch(id) } }.awaitAll()
}

suspend fun fireMany(items: List<Item>) = coroutineScope {
    items.map { launch { process(it) } }.joinAll()
}
```

`awaitAll` — fail-fast, любой fail отменяет остальных и бросает.

### Когда launch, когда async, когда withContext

- **launch**: fire-and-forget, нет результата (логирование, обновление БД без UI).
- **async**: нужен результат + параллельность с другими.
- **withContext**: смена dispatcher, нужно дождаться результата, последовательность.
- **runBlocking**: только main/тесты/Java-interop.

### async в for-loop

```kotlin
// ❌ запускается по одной (нет параллельности — сразу await)
for (id in ids) {
    val x = async { fetch(id) }.await()
}

// ✅
val results = ids.map { async { fetch(it) } }.awaitAll()
```

## Подводные камни

- **`async` без обёртки в coroutineScope** — exception теряется до await.
- **Последовательные `withContext`** там, где должны быть параллельные `async`.
- **`runBlocking` на Main** — потенциальный deadlock.
- **`async` для fire-and-forget** — теряется Deferred, exception игнорируется. Лучше launch.
- **`withContext(Dispatchers.IO)` поверх suspend Retrofit/Room** — избыточно, они уже main-safe.
- **`launch().join()` вместо `withContext`** — тот же результат, но запутывает читающего.

## Связанные темы

- [[q-02-coroutine-scope-context]]
- [[q-03-dispatchers]]
- [[q-04-structured-concurrency]]

## Follow-up

- Чем `withContext` отличается от `async + await`?
- Когда `async`, когда `launch`?
- Почему `runBlocking` опасен на Main thread?
