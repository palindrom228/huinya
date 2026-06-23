---
type: question
category: concurrency
difficulty: senior
tags: [coroutines, exceptions, handler]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# CoroutineExceptionHandler, обработка исключений в корутинах

## Краткий ответ (TL;DR)

Exception в `launch` → propagates до **root** scope с `SupervisorJob` или `supervisorScope`, попадает в `CoroutineExceptionHandler`. Без handler — crash (для launch) / silent (для async до await). Handler — только в **root** корутине (или прямом ребёнке supervisor). В обычной иерархии Job — child exception отменяет parent и siblings. Не путать с `try/catch` — handler ловит **uncaught**, try/catch внутри корутины обрабатывает явно.

## Развёрнутый ответ

### CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    Timber.e(exception, "Uncaught in ${context[CoroutineName]?.name}")
}

val scope = CoroutineScope(SupervisorJob() + handler + Dispatchers.Default)

scope.launch {
    throw RuntimeException("boom")
}
// → handler логирует, scope продолжает работать
```

### Где handler работает

- ✅ Root launch на scope с SupervisorJob.
- ✅ Прямой child supervisorScope.
- ❌ Вложенный launch внутри другого launch (exception propagates до root).
- ❌ async — exception хранится в Deferred до await.

```kotlin
scope.launch(handler) {           // root — handler сработает
    launch {                      // child — exception propagates до root
        throw RuntimeException()
    }
}
```

### async поведение

```kotlin
val d = scope.async { throw RuntimeException() }
// handler НЕ срабатывает сразу

try {
    d.await()                     // здесь throws
} catch (e: Exception) {
    handle(e)
}
```

В обычном Job exception от async **всё равно** отменяет parent. В supervisorScope — нет.

### Что делать в Android

#### viewModelScope

```kotlin
class MyVm : ViewModel() {
    private val handler = CoroutineExceptionHandler { _, e ->
        Timber.e(e, "uncaught in VM")
        Firebase.crashlytics().recordException(e)
        _state.value = UiState.Error(e.toUserMessage())
    }

    fun load() = viewModelScope.launch(handler) {
        repo.fetch()
    }
}
```

`viewModelScope` уже SupervisorJob → handler работает на root launch.

#### Application-wide handler

```kotlin
@HiltAndroidApp
class App : Application() {
    @Inject lateinit var crashlytics: CrashlyticsLogger

    val handler = CoroutineExceptionHandler { _, e ->
        crashlytics.log(e)
    }

    val appScope = CoroutineScope(SupervisorJob() + handler + Dispatchers.Default)
}
```

### try/catch внутри корутины

```kotlin
viewModelScope.launch {
    try {
        repo.fetch()
    } catch (c: CancellationException) {
        throw c
    } catch (e: IOException) {
        _state.value = UiState.Error("Нет сети")
    }
}
```

Предпочтительный способ для **business errors**. Handler — для **uncaught/unexpected**.

### runCatching pitfall

```kotlin
runCatching { repo.fetch() }
    .onSuccess { ... }
    .onFailure { ... }   // ❌ ловит CancellationException!
```

Safer:

```kotlin
inline fun <T> runCatchingCancellable(block: () -> T): Result<T> =
    try { Result.success(block()) }
    catch (c: CancellationException) { throw c }
    catch (t: Throwable) { Result.failure(t) }
```

### Default uncaught handler

Без `CoroutineExceptionHandler`:
- `launch` → системный `Thread.uncaughtExceptionHandler` (обычно crash).
- `GlobalScope.launch` — то же.

На Android при `launch` без handler — приложение упадёт (Firebase Crashlytics зарегистрирует).

### Иерархия propagation

```
                Root SupervisorJob (handler)
                       │
                ┌──────┴──────┐
                │             │
            launch1       launch2
              │
        ┌─────┴─────┐
        │           │
    inner1      inner2 (throws)
```

inner2 throws → отменяет launch1 и inner1 (нет supervisor) → exception propagates до Root → handler.

С `supervisorScope` в launch1:

```
            launch1 = supervisorScope
                │
        ┌───────┴────────┐
        │                │
    inner1          inner2 (throws)
```

inner2 throws → НЕ отменяет inner1, но ОН САМ упал → exception в handler launch1 (если есть) или **выше**.

### Multi-handler — последний выигрывает

```kotlin
val h1 = CoroutineExceptionHandler { _, _ -> println("h1") }
val h2 = CoroutineExceptionHandler { _, _ -> println("h2") }

scope.launch(h1) {
    launch(h2) { throw RuntimeException() }
}
// h2 не сработает (вложенный child), h1 сработает на root
```

### Особый случай: SupervisorJob в child

```kotlin
val outer = CoroutineScope(Job() + handler)
outer.launch {
    val inner = CoroutineScope(SupervisorJob())
    inner.launch { throw RuntimeException() }
    // inner.handler не задан → crash
}
```

Внутренние scope требуют своего handler.

### Logging уровень

```kotlin
val handler = CoroutineExceptionHandler { ctx, e ->
    when (e) {
        is IOException -> Timber.w(e, "network")
        is HttpException -> Timber.w(e, "http ${e.code()}")
        else -> {
            Timber.e(e, "unexpected")
            Crashlytics.recordException(e)
        }
    }
}
```

## Подводные камни

- Handler **только в root** — навешивать на каждый child бесполезно.
- `async` exception потеряется без await → silent failures.
- `catch (Exception)` без re-throw `CancellationException` → ломает structured concurrency.
- Без handler в `GlobalScope.launch` → crash приложения.
- Handler ловит uncaught, не "все" — try/catch внутри отрабатывает свои exceptions раньше.
- Хочешь "глобально логировать всё" → добавь handler в `viewModelScope` и в `Application` scope, не везде.

## Связанные темы

- [[q-04-structured-concurrency]]
- [[q-05-supervisor-job]]
- [[q-06-cancellation]]
- [[../04-Architecture/q-15-error-handling]]

## Follow-up

- Где работает `CoroutineExceptionHandler`?
- Почему `async` exception тихий?
- Чем handler отличается от try/catch?
