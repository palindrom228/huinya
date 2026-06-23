---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, supervisor, job]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# SupervisorJob, supervisorScope: когда независимые children

## Краткий ответ (TL;DR)

Обычный `Job`: fail одного child → cancel parent → cancel siblings. `SupervisorJob`: fail child **не отменяет** parent/siblings. Используется в (1) **long-lived scopes** (ViewModel, repository scope) — чтобы одна упавшая корутина не убила весь scope, (2) **independent parallel tasks** — где допустимы частичные результаты. `supervisorScope { }` = `coroutineScope` с supervisor поведением.

## Развёрнутый ответ

### Различие наглядно

```kotlin
// Обычный Job
val scope1 = CoroutineScope(Job())
scope1.launch {
    launch { throw RuntimeException("boom") }
    launch { delay(1000); println("never") }  // отменится
}
delay(2000)
println(scope1.isActive)  // false — весь scope мёртв

// SupervisorJob
val scope2 = CoroutineScope(SupervisorJob())
scope2.launch { throw RuntimeException("boom") }
scope2.launch { delay(1000); println("printed") }  // продолжит работать
println(scope2.isActive)  // true
```

### viewModelScope использует SupervisorJob

```kotlin
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) return scope
        return setTagIfAbsent(JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate))
    }
```

Поэтому одна упавшая `viewModelScope.launch { }` не отменяет другие.

### supervisorScope

```kotlin
suspend fun loadAll(urls: List<String>) = supervisorScope {
    urls.map { url ->
        launch {
            try { downloadOne(url) }
            catch (e: Exception) { logError(e) }
        }
    }
}
```

**Важно**: внутри `supervisorScope` всё ещё нужен `try/catch` в child — иначе uncaught exception попадёт в `CoroutineExceptionHandler` (или crash, если нет handler).

### async в supervisorScope

```kotlin
suspend fun loadBoth() = supervisorScope {
    val a = async { fetchA() }   // exception хранится в Deferred
    val b = async { fetchB() }

    val resultA = try { a.await() } catch (e: Exception) { null }
    val resultB = try { b.await() } catch (e: Exception) { null }

    resultA to resultB
}
```

В обычном `coroutineScope` exception в `async` всё равно отменил бы siblings до `await()`.

### Custom scope с SupervisorJob

```kotlin
class AppScope @Inject constructor() {
    val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
}

// Repository:
class SyncRepository @Inject constructor(private val appScope: AppScope) {
    fun syncInBackground() = appScope.scope.launch {
        // одна упавшая sync не убьёт другие
    }
}
```

Application-scoped operations, которые переживают экраны.

### CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    Timber.e(e, "Uncaught coroutine exception")
}

val scope = CoroutineScope(SupervisorJob() + handler)

scope.launch { throw RuntimeException() }  // попадёт в handler
```

Handler работает **только** в root корутине scope (с SupervisorJob) или в прямых детях `supervisorScope`. Не в `async`.

### Когда использовать обычный Job

- Сильная связность задач: если одна упала — остальные бессмысленны.
- Пример: `coroutineScope { val user = async { fetchUser() }; val profile = async { fetchProfile() }; combine(user.await(), profile.await()) }` — если user не загрузился, profile бесполезен.

### Когда SupervisorJob

- Long-lived scope (ViewModel, Application, Singleton Repository).
- Параллельные independent задачи (загрузка списка изображений).
- "Background job manager" — каждая job изолирована.

### Подводный камень: child Job в Supervisor

```kotlin
val scope = CoroutineScope(SupervisorJob())
scope.launch {                          // child1 — Job, parent = SupervisorJob
    launch { throw RuntimeException() } // child1.1 — Job, parent = child1
    launch { delay(1000) }              // child1.2 — отменится!
}
```

SupervisorJob применяется **только к прямым детям**. `child1` — обычный Job, внутри него обычное поведение. Чтобы дети child1 тоже были независимы — нужен `supervisorScope { }` внутри.

### supervisorScope vs SupervisorJob

| | supervisorScope | SupervisorJob |
|--|-----------------|---------------|
| Тип | suspend builder | Job-элемент |
| Использование | внутри suspend функции | в CoroutineScope() |
| Lifetime | scope блока | пока scope активен |
| Аналог | coroutineScope { } | Job() |

### Пример: загрузка батча с retries

```kotlin
suspend fun loadBatch(ids: List<Long>): List<Result<Item>> = supervisorScope {
    ids.map { id ->
        async {
            runCatchingCancellable {
                retryWithBackoff { api.fetch(id) }
            }
        }
    }.awaitAll()
}
```

Каждый item загружается независимо, fail одного не валит batch.

## Подводные камни

- `SupervisorJob` НЕ ловит exceptions — нужен `CoroutineExceptionHandler` или try/catch в child.
- `supervisorScope { val x = async { fail() }; x.await() }` — `await()` бросит exception. Supervisor спасает siblings, но не сам caller.
- Supervisor только для прямых детей — внуки следуют обычным правилам, нужен вложенный `supervisorScope`.
- Использование Supervisor "на всякий случай" → можно проглотить баги, которые должны были crash приложение.
- `SupervisorJob().cancel()` всё равно отменяет всех детей — supervisor про exceptions, не про cancellation.

## Связанные темы

- [[q-04-structured-concurrency]]
- [[q-06-cancellation]]
- [[q-08-exception-handler]]

## Follow-up

- Чем `SupervisorJob` отличается от обычного `Job`?
- Почему `viewModelScope` использует SupervisorJob?
- Когда supervisor поведение неуместно?
