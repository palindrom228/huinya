---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines, channels]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# `Channel` vs `Flow`: когда что? `callbackFlow`, `channelFlow`

## Краткий ответ (TL;DR)

**`Channel`** — рандеву между корутинами, **hot, one-shot** (значение получает один коллектор). **`Flow`** — декларативный pipeline, **cold by default**, может броадкаститься через `SharedFlow`. Для адаптации callback-API → `callbackFlow` (concurrent emit OK) или `flow { }` (один продюсер).

## Развёрнутый ответ

### Channel

```kotlin
val ch = Channel<Int>(capacity = Channel.BUFFERED)
launch { ch.send(1); ch.send(2); ch.close() }
launch { for (x in ch) println(x) }
```

- **Producer-consumer** примитив.
- Значение получает **один** consumer (если несколько — каждый получает свой кусок, не дубль).
- Закрывается явно (`close`).
- Capacity: `RENDEZVOUS`, `CONFLATED`, `UNLIMITED`, `BUFFERED`, или конкретное число.
- При полном буфере `send` suspend-ит, `trySend` возвращает result.

### Flow

- **Декларативный**, операторы.
- **Cold**: производство стартует по collect.
- **One-to-many**: каждый collect — отдельный «прогон» (для cold). Для shared — `SharedFlow`/`StateFlow`.
- Закрывается по завершению блока или exception.

### callbackFlow

Идеален для адаптации callback-API (Firebase, location, broadcast):

```kotlin
fun locationFlow(client: FusedLocationProviderClient): Flow<Location> = callbackFlow {
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }
    client.requestLocationUpdates(request, callback, Looper.getMainLooper())
    awaitClose { client.removeLocationUpdates(callback) }
}
```

- Внутри `Channel` под капотом (поэтому можно эмитить из разных потоков).
- Обязательно `awaitClose { /* cleanup */ }` — иначе утечка.
- Backpressure через capacity (`callbackFlow(capacity = BUFFERED)`), по умолчанию RENDEZVOUS → блокирует callback.

### channelFlow

Когда нужно concurrent emit из нескольких корутин:

```kotlin
fun combined(): Flow<Result> = channelFlow {
    launch { repeat(10) { send(api1()) } }
    launch { repeat(10) { send(api2()) } }
}
```

Внутри — `Channel`, поэтому можно `send` из дочерних корутин (в обычном `flow { }` это запрещено — `EmissionException`).

### Когда что выбрать

| Кейс | Что |
|------|-----|
| Адаптировать callback-based API | `callbackFlow` |
| Concurrent эмит из нескольких источников | `channelFlow` |
| Один продюсер, последовательный | `flow { }` |
| Стейт UI | `StateFlow` |
| One-shot события (snackbar) | `SharedFlow(replay = 0)` или Channel |
| Pipeline процессинга задач | Channel |

### Channel vs SharedFlow для событий

Долгое время Channel был стандартом для one-shot events. Сейчас часто используют `SharedFlow`:

| | Channel | SharedFlow |
|---|---------|------------|
| Принимает | один collector | много collectors |
| Сложно собрать дважды (consume-once) | да | нет |
| Подходит для «event bus» | сложнее | проще |
| `repeatOnLifecycle` корректно | внимательно, **события теряются** при остановке collect | теряются, если `replay = 0` |

⚠️ Главная проблема обоих: пока подписки нет, события могут теряться. Решение — либо буфер, либо «события как часть state».

## Подводные камни

- `callbackFlow` без `awaitClose` → утечка callback и ресурса.
- `Channel.send` блокирующий (suspend) — если не учли, можно ловить ANR-подобные эффекты (не в UI, но в pipeline).
- Закрытый Channel при попытке `send` бросает `ClosedSendChannelException`.
- В `flow { }` нельзя `emit` из другого корутины — `IllegalStateException`. Используй `channelFlow`.

## Связанные темы

- [[q-13-flow-basics]]
- [[q-12-cancellation-exceptions]]

## Follow-up

- Чем отличается `RENDEZVOUS` channel от `BUFFERED`?
- Почему `flow { }` не позволяет `launch { emit() }` внутри?
