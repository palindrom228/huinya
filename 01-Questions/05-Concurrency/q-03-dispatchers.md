---
type: question
category: concurrency
difficulty: middle
tags: [coroutines, dispatchers, threading]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Dispatchers: Main / IO / Default / Unconfined — когда что

## Краткий ответ (TL;DR)

Dispatcher определяет поток (или пул), на котором выполняется continuation. **Main** — UI thread (один). **Default** — CPU-bound, пул = `max(2, кол-во ядер)`. **IO** — blocking I/O, пул до 64 (расширяется поверх Default). **Unconfined** — без переключения (текущий поток вызывающего), для тестов / lightweight операций. `Main.immediate` — оптимизация, если уже на Main, не диспатчит. Custom через `Executor.asCoroutineDispatcher()`.

## Развёрнутый ответ

### Dispatchers.Main

Один UI поток.

```kotlin
viewModelScope.launch {        // Main по умолчанию
    val data = withContext(Dispatchers.IO) { repo.load() }
    _state.value = data        // обратно на Main
}
```

Требует `kotlinx-coroutines-android` (содержит реализацию). Без неё — `IllegalStateException`.

### Dispatchers.Main.immediate

```kotlin
withContext(Dispatchers.Main.immediate) { /* ... */ }
```

Если уже на Main → выполняется сразу без диспатча (no overhead). Если на другом потоке — постит в Main queue. Полезно для частых вызовов, которые **обычно** уже на Main.

### Dispatchers.Default

CPU-bound работа: JSON parsing, сортировка, image processing.

```kotlin
val sorted = withContext(Dispatchers.Default) {
    bigList.sortedBy { it.score }
}
```

Размер пула = `max(2, Runtime.availableProcessors())`. Шаринг с другими корутинами Default.

### Dispatchers.IO

Blocking I/O: файлы, сеть (без OkHttp async), Room.

```kotlin
val text = withContext(Dispatchers.IO) {
    File("data.txt").readText()
}
```

Лимит — 64 потока (или больше при `kotlinx.coroutines.io.parallelism`). **Шарит** потоки с Default — при переключении Default↔IO может остаться на том же потоке (оптимизация).

### Dispatchers.Unconfined

Без диспатча — выполняется на потоке, который вызвал resume.

```kotlin
withContext(Dispatchers.Unconfined) {
    println(Thread.currentThread().name)  // зависит от того, кто резюмировал
    delay(100)
    println(Thread.currentThread().name)  // изменилось!
}
```

Полезно: тесты (`Dispatchers.Unconfined` = всё синхронно через `runBlockingTest`), очень лёгкие операции. **Опасно** для UI-кода — непредсказуемый поток.

### Main-safety pattern

Suspend функция должна быть **main-safe** — не блокировать Main thread:

```kotlin
// ✅ main-safe
class UserRepository @Inject constructor(private val api: UserApi) {
    suspend fun getUser(id: Long): User = withContext(Dispatchers.IO) {
        api.fetchUserBlocking(id)  // blocking call, обёрнут в IO
    }
}

// VM может вызвать без withContext:
val user = userRepo.getUser(id)
```

### Custom dispatcher

```kotlin
val singleThread = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
val custom = Executors.newFixedThreadPool(4).asCoroutineDispatcher()

withContext(singleThread) { /* всё последовательно */ }

// ВАЖНО: close после использования
singleThread.close()
```

Используй для:
- Сериализованного доступа к ресурсу (один поток).
- Изоляции от Default/IO (отдельная очередь для critical).

### limitedParallelism (Kotlin 1.6+)

```kotlin
val limited = Dispatchers.IO.limitedParallelism(8)
withContext(limited) { /* максимум 8 параллельных */ }
```

Не создаёт новых потоков — ограничивает параллелизм поверх существующего пула. Заменяет custom dispatcher для большинства случаев.

### Inject dispatchers (testing)

```kotlin
class NewsRepository @Inject constructor(
    private val api: NewsApi,
    @IoDispatcher private val io: CoroutineDispatcher
) {
    suspend fun fetch() = withContext(io) { api.get() }
}

// В тестах:
val repo = NewsRepository(fakeApi, StandardTestDispatcher())
```

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @IoDispatcher @Provides
    fun io(): CoroutineDispatcher = Dispatchers.IO

    @DefaultDispatcher @Provides
    fun default(): CoroutineDispatcher = Dispatchers.Default

    @MainDispatcher @Provides
    fun main(): CoroutineDispatcher = Dispatchers.Main
}
```

### Когда что использовать

| Задача | Dispatcher |
|--------|------------|
| Обновить UI / state | Main |
| Часто вызываемый Main-coroutine | Main.immediate |
| JSON parsing, сортировка, шифрование | Default |
| File I/O, JDBC, Room blocking, OkHttp execute() | IO |
| Тесты (legacy), очень короткие операции | Unconfined |
| Сериализация доступа | limitedParallelism(1) или single-thread executor |

### Retrofit / OkHttp suspend

Retrofit suspend методы — уже main-safe. **Не оборачивай в withContext(IO)** ещё раз. OkHttp использует свой пул потоков для запросов.

```kotlin
// ✅
suspend fun fetch(): User = api.getUser()   // Retrofit handles dispatching

// ❌ избыточно
suspend fun fetch(): User = withContext(Dispatchers.IO) { api.getUser() }
```

## Подводные камни

- **Dispatchers.IO для CPU-bound** — IO предназначен для blocking I/O, не для тяжёлых вычислений. Тяжёлый JSON парсинг на IO забьёт пул.
- **Default для blocking IO** — мало потоков, заблокированный поток = меньше параллелизма для других CPU-задач.
- **`runBlocking` на Main** — deadlock на старте корутины.
- **Custom dispatcher без close** — leak потоков пула.
- **Unconfined в production коде** — корутина может оказаться на random thread после resume → race condition.
- **Двойное withContext(IO)** в Retrofit/Room — избыточно, они уже main-safe.
- **Main.immediate в виде "оптимизации везде"** — может скрыть проблемы (что-то должно было быть async, но выполнилось sync).

## Связанные темы

- [[q-01-suspend-under-the-hood]]
- [[q-02-coroutine-scope-context]]
- [[q-18-threads-handler-looper]]

## Follow-up

- Чем `Default` отличается от `IO`?
- Что такое `Main.immediate` и когда его использовать?
- Зачем инжектить Dispatcher через DI?
