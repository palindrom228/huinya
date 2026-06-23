---
type: question
category: kotlin
difficulty: middle
tags: [kotlin, error-handling]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# `Result`, `runCatching`, обработка ошибок без exceptions

## Краткий ответ (TL;DR)

`Result<T>` — value class-обёртка для success/failure. `runCatching { ... }` ловит все `Throwable` и возвращает `Result`. Удобно для функциональной композиции, но **с осторожностью в корутинах** — глотает `CancellationException`.

## Развёрнутый ответ

### Result

```kotlin
val r: Result<User> = runCatching { api.fetchUser() }
val user: User = r.getOrNull() ?: defaultUser
val userOrThrow: User = r.getOrThrow()
val mapped: Result<String> = r.map { it.name }
val recovered: Result<User> = r.recover { defaultUser }
r.onSuccess { ... }.onFailure { Log.e(...) }
```

`Result<T>` — `@JvmInline value class` над `Any?`: либо `T`, либо обёртка с исключением. Без аллокации на success (если не value class дополнительно боксится в дженериках).

### runCatching внутри корутин — опасно

```kotlin
suspend fun load(): Result<User> = runCatching { api.fetch() }
```

Если корутина отменяется во время `api.fetch()` → `CancellationException` → `runCatching` поймает её как «обычную ошибку» → корутина не остановится, scope получит сломанный state.

**Правило:**

```kotlin
return runCatching { api.fetch() }
    .onFailure { if (it is CancellationException) throw it }
```

Или собственная обёртка:

```kotlin
suspend inline fun <R> coRunCatching(block: () -> R): Result<R> = try {
    Result.success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: Throwable) {
    Result.failure(e)
}
```

### Result как возвращаемый тип

Изначально стдлиб **запрещал** `Result` как тип параметра/возвращаемого значения публичных функций — чтобы избежать перегрузок Either-like. Сейчас (Kotlin 1.5+) — разрешён, но всё ещё спорно: лучше доменные `sealed interface` («Result.Success/Error» под свой домен).

### Альтернативы

```kotlin
sealed interface UserResult {
    data class Success(val user: User) : UserResult
    data class Error(val type: ErrorType, val cause: Throwable? = null) : UserResult
}
```

- Явная семантика типов ошибок.
- Exhaustive `when`.
- Не глотает Cancellation.

Библиотеки: **Arrow's `Either<L, R>`**, **kotlin-result**.

### Когда что

| | `try/catch` | `Result/runCatching` | sealed result |
|---|-------------|---------------------|----------------|
| Простой императивный код | ✅ | | |
| Композиция (`map`, `flatMap`) | | ✅ | можно (`map`) |
| Доменная семантика ошибок | | | ✅ |
| Корутины (отмена) | ✅ | ⚠️ требует CancellationException check | ✅ |

## Подводные камни

- `runCatching` ловит **всё** включая `OutOfMemoryError`, `CancellationException`, `Error`. Часто хочется только `Exception`.
- `Result.failure(Throwable())` без причины — теряется стек.
- Returning `Result` из API → пользователи могут забыть проверить failure.
- `Result == Result` сравнивает по значению, но stack trace отличается → не используй в equals-сценариях.

## Связанные темы

- [[q-12-cancellation-exceptions]]
- [[q-04-data-vs-value-vs-sealed]]

## Follow-up

- Почему `runCatching` опасен внутри корутин?
- Чем `Result<T>` лучше/хуже доменного `sealed interface`?
