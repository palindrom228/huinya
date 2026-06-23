---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Как устроены корутины под капотом? `suspend`, continuation, state machine

## Краткий ответ (TL;DR)

`suspend`-функция компилируется в обычный метод с дополнительным параметром `Continuation`. Внутри тело превращается в **state machine**: каждая точка suspend — отдельное состояние. При suspend функция возвращает `COROUTINE_SUSPENDED`, состояние сохраняется в continuation, а корутина продолжается, когда `Continuation.resume()` будет вызван (например, из callback'а).

## Развёрнутый ответ

### Continuation-passing style (CPS)

```kotlin
suspend fun fetch(): User { ... }
```

Компилируется примерно как:

```java
Object fetch(Continuation<? super User> $cont) { ... }
```

Когда корутина вызывает другую suspend-функцию — она передаёт continuation. Если функция готова сразу — возвращает значение. Если нет — возвращает `COROUTINE_SUSPENDED`, а в нужный момент кто-то вызывает `$cont.resumeWith(Result.success(user))`.

### State machine

```kotlin
suspend fun login(): User {
    val token = api.auth()     // state 1
    val user = api.me(token)   // state 2
    return user                // state 3
}
```

Компилятор разбивает на состояния по точкам suspend:

```java
switch (state) {
  case 0:
    state = 1; Object t = api.auth($cont);
    if (t == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED;
    // fall through
  case 1:
    token = (String) t;
    state = 2; Object u = api.me(token, $cont);
    if (u == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED;
  case 2:
    user = (User) u;
    return user;
}
```

Локальные переменные хранятся как поля continuation-объекта.

### Запуск корутины

```kotlin
scope.launch {        // строит корневой Job
    val u = fetch()
    show(u)
}
```

`launch` принимает `suspend` лямбду, оборачивает в continuation, запускает на диспатчере. `Dispatcher` определяет, в каком потоке выполнять resume.

### Структурная конкурентность

Корневой `Job` родительский для всех корутин внутри. `scope.cancel()` отменяет всех детей. Исключение в ребёнке отменяет родителя (кроме `SupervisorJob`).

### Главные понятия

- **`CoroutineScope`** — контекст + Job, привязывает корутины к жизненному циклу.
- **`CoroutineContext`** — мапа элементов: `Job`, `Dispatcher`, `Name`, `ExceptionHandler`.
- **`Job`** — handle к корутине: `cancel`, `join`, `children`.
- **`Dispatcher`** — Main, Default (CPU), IO, Unconfined.
- **`withContext`** — переключает контекст, ждёт результата.
- **`async/await`** — параллельный fork-join, `Deferred<T>`.

## Подводные камни

- `suspend` ≠ async/non-blocking. Это просто маркер «может приостановиться». Если внутри `Thread.sleep` — она блокирует поток.
- Resume может прийти на другом потоке (зависит от dispatcher) — никакого ThreadLocal без `withContext`.
- Лишние `withContext(Dispatchers.IO)` поверх suspend-репозитория — анти-паттерн (главное правило: «suspend-функции должны быть main-safe», но это ответственность нижнего слоя).
- Стек корутин **не** Java stack — для дебага нужны `kotlinx-coroutines-debug` / IDE настройки.

## Связанные темы

- [[q-11-coroutine-context]]
- [[q-12-cancellation-exceptions]]
- [[q-13-flow-basics]]

## Follow-up

- Что произойдёт, если внутри `suspend` функции вызвать `Thread.sleep`?
- Чем `coroutineScope { }` отличается от `supervisorScope { }`?
- Где живут локальные переменные suspend-функции на момент suspend?
