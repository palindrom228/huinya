---
type: question
category: architecture
difficulty: senior
tags: [error-handling, result, exceptions]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Error handling в архитектуре: Result vs Exception, retry, UX

## Краткий ответ (TL;DR)

Два стиля: **throws** (suspend функция бросает, поймать через try/catch или `runCatching`) или **typed Result** (sealed class / kotlin.Result / arrow.Either). Throws естественнее для cancellation/system errors, Result удобнее для business errors с типизацией. На UI уровне — sealed `UiState.Error` или `errorMessage` в state. Retry — `retry` operator + `WhileSubscribed` + exponential backoff.

## Развёрнутый ответ

### Throws подход

```kotlin
suspend fun getUser(id: Long): User = withContext(io) {
    api.fetchUser(id)   // может бросить IOException, HttpException
}

// VM:
fun load() = viewModelScope.launch {
    _state.value = UiState.Loading
    try {
        val user = repo.getUser(id)
        _state.value = UiState.Success(user)
    } catch (e: IOException) {
        _state.value = UiState.Error("Нет интернета")
    } catch (e: HttpException) {
        _state.value = UiState.Error("Server error: ${e.code()}")
    }
}
```

Плюсы: естественно с suspend, cancellation работает (CancellationException не ловится `try/catch` без re-throw).

### runCatching

```kotlin
fun load() = viewModelScope.launch {
    runCatching { repo.getUser(id) }
        .onSuccess { _state.value = UiState.Success(it) }
        .onFailure { _state.value = UiState.Error(it.toUserMessage()) }
}
```

**Внимание**: `runCatching` ловит `Throwable` включая **CancellationException** → корутина не отменяется! Опасно. Исправление:

```kotlin
inline fun <T> runCatchingCancellable(block: () -> T): Result<T> =
    try { Result.success(block()) }
    catch (c: CancellationException) { throw c }
    catch (t: Throwable) { Result.failure(t) }
```

С Kotlin 1.9+ есть `coroutineScope { ... }` который правильно обрабатывает.

### Typed Result (sealed)

```kotlin
sealed interface Outcome<out T> {
    data class Success<T>(val value: T) : Outcome<T>
    sealed interface Failure : Outcome<Nothing> {
        data object Network : Failure
        data class Server(val code: Int) : Failure
        data class Unknown(val cause: Throwable) : Failure
    }
}

suspend fun getUser(id: Long): Outcome<User> = try {
    Outcome.Success(api.fetchUser(id))
} catch (e: IOException) { Outcome.Failure.Network }
  catch (e: HttpException) { Outcome.Failure.Server(e.code()) }
  catch (c: CancellationException) { throw c }
  catch (t: Throwable) { Outcome.Failure.Unknown(t) }
```

Плюсы: компилятор enforce'ит обработку всех cases. Типизированные ошибки.
Минусы: boilerplate, нужно решать что считать ошибкой vs exception.

### Arrow Either

```kotlin
import arrow.core.Either
suspend fun getUser(id: Long): Either<UserError, User> = either {
    val dto = networkCall().bind()
    val parsed = parse(dto).bind()
    parsed
}
```

Функциональный стиль, monad-like composition. Хорош в KMP проектах.

### UI представление

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Data(val user: User) : UiState
    data class Error(val message: String, val canRetry: Boolean) : UiState
}

@Composable
fun UserScreen(vm: UserVm) {
    val state by vm.state.collectAsStateWithLifecycle()
    when (state) {
        is UiState.Loading -> ProgressIndicator()
        is UiState.Data -> UserCard(state.user)
        is UiState.Error -> ErrorView(state.message, onRetry = vm::retry.takeIf { state.canRetry })
    }
}
```

### Retry с backoff

```kotlin
fun <T> Flow<T>.retryWithBackoff(
    maxAttempts: Int = 3,
    initialDelay: Long = 1000
): Flow<T> = retry(maxAttempts.toLong()) { e ->
    if (e is IOException) {
        delay(initialDelay * (1 shl attempt))   // exponential
        true
    } else false
}
```

Или ручной цикл:

```kotlin
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 1000,
    block: suspend () -> T
): T {
    var delay = initialDelay
    repeat(times - 1) {
        runCatching { return block() }
        delay(delay)
        delay *= 2
    }
    return block()  // last try, throws на failure
}
```

### Mapping technical → user errors

```kotlin
fun Throwable.toUserMessage(): String = when (this) {
    is UnknownHostException -> "Нет соединения с сервером"
    is SocketTimeoutException -> "Превышено время ожидания"
    is HttpException -> when (code()) {
        401 -> "Сессия истекла"
        403 -> "Нет доступа"
        in 500..599 -> "Сервер недоступен"
        else -> "Ошибка сети"
    }
    else -> "Что-то пошло не так"
}
```

Никогда не показывай raw stack trace пользователю.

### Logging

```kotlin
.onFailure { error ->
    Timber.e(error, "getUser failed")
    Firebase.crashlytics().recordException(error)
}
```

Логирование — отдельный слой (interceptor / decorator), не разбросанные `Timber.e()`.

## Подводные камни

- `runCatching` без перехвата `CancellationException` → корутина не отменяется.
- `try/catch(Exception)` поглощает `CancellationException` (раньше до Kotlin 1.5 чаще; сейчас IDE предупреждает).
- Error как `String?` в state и забыли сбросить → snackbar показывается снова после rotation. См. [[q-05-side-effects]].
- Слишком общий catch → проглатываем баги (NPE → "что-то пошло не так").
- `Result<T>` в Repository, который возвращает Flow → каждый emit Result значит много boilerplate. Часто проще throws + try в VM.

## Связанные темы

- [[q-04-udf]]
- [[q-09-repository-pattern]]
- [[../05-Concurrency/q-coroutine-cancellation]]

## Follow-up

- Чем опасен `runCatching` в suspend контексте?
- Когда использовать typed Result, а когда throws?
- Как реализовать exponential backoff для retry?
