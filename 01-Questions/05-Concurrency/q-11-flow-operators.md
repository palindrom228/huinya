---
type: question
category: concurrency
difficulty: senior
tags: [flow, operators, flatmap, buffer, debounce]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Flow operators: flatMap{Merge,Concat,Latest}, buffer, conflate, debounce

## Краткий ответ (TL;DR)

**flatMapConcat** — sequential, ждёт завершения предыдущего. **flatMapMerge** — параллельно (concurrency лимит). **flatMapLatest** — отменяет предыдущий при новом emit. **buffer** — async producer/consumer, буфер между ними. **conflate** — выкидывает промежуточные значения. **debounce** — эмитит только если паузу N мс не было новых. **sample** — последнее значение раз в N мс. **distinctUntilChanged** — пропускает дубликаты.

## Развёрнутый ответ

### Transformation основные

```kotlin
flow.map { it * 2 }
flow.filter { it > 0 }
flow.transform { value ->
    emit("a $value")
    emit("b $value")
}
flow.onEach { log(it) }
```

### flatMap варианты

```kotlin
// flatMapConcat — sequential
flowOf(1, 2, 3).flatMapConcat { i ->
    flow { emit(i); delay(100); emit(i * 10) }
}
// → 1, 10, 2, 20, 3, 30 (последовательно)

// flatMapMerge — concurrent
flowOf(1, 2, 3).flatMapMerge(concurrency = 2) { i ->
    flow { delay(100); emit(i) }
}
// → 1, 2, 3 (параллельно, порядок может варьироваться)

// flatMapLatest — отмена предыдущего
flowOf(1, 2, 3).flatMapLatest { i ->
    flow { delay(100); emit(i) }
}
// → 3 (1, 2 отменены до завершения)
```

Типичный use case `flatMapLatest`: поиск по тексту — новый запрос отменяет предыдущий.

```kotlin
searchQuery
    .debounce(300)
    .flatMapLatest { query -> api.search(query) }
    .collect { results -> _state.value = results }
```

### buffer

```kotlin
flow {
    repeat(3) { emit(it); delay(100) }
}
    .map { delay(300); it * 2 }   // slow consumer
    .collect { println(it) }
```

Без buffer: ~900ms (100+300, 100+300, 100+300).
С `.buffer()`: emit и consume параллельно — ~700ms.

```kotlin
flow.buffer(capacity = 64, onBufferOverflow = BufferOverflow.DROP_OLDEST)
```

### conflate

```kotlin
flow {
    repeat(100) { emit(it); delay(10) }
}.conflate().collect { delay(500); println(it) }
// → 0, 99 (промежуточные потеряны)
```

`.conflate()` = `.buffer(capacity = CONFLATED)`.

### debounce / sample

```kotlin
searchQuery
    .debounce(300)    // эмитит после 300мс без новых
    .filter { it.isNotBlank() }

stream
    .sample(1000)     // последнее значение раз в 1 сек
```

### distinctUntilChanged

```kotlin
flow.distinctUntilChanged()
flow.distinctUntilChanged { old, new -> old.id == new.id }
```

StateFlow и так distinct, добавлять обычно не нужно.

### combine / zip / merge

```kotlin
// combine — каждый emit любого источника → пересчитать
combine(userFlow, settingsFlow) { user, settings ->
    Profile(user, settings)
}

// zip — пара emit'ов из обоих (медленнее догоняет)
flow1.zip(flow2) { a, b -> a to b }

// merge — все эмишены в один flow
merge(flow1, flow2, flow3)
```

### onStart / onCompletion / catch

```kotlin
flow
    .onStart { emit(Loading) }
    .onCompletion { error ->
        if (error == null) log("finished")
        else log("failed: $error")
    }
    .catch { e ->
        emit(ErrorState(e))   // обработка upstream exceptions
    }
    .collect { _state.value = it }
```

`catch` — ловит **upstream** exceptions, не downstream. Для downstream (включая collector) — try/catch вокруг collect.

### retry / retryWhen

```kotlin
flow.retry(3) { it is IOException }

flow.retryWhen { cause, attempt ->
    if (cause is IOException && attempt < 3) {
        delay(1000 * (1 shl attempt.toInt()))
        true
    } else false
}
```

### take / takeWhile / first / single

```kotlin
flow.take(5)               // первые 5
flow.takeWhile { it < 10 } // пока true

flow.first()               // первое и cancel
flow.first { it > 0 }      // первое подходящее
flow.single()              // ровно одно (иначе exception)
```

### scan / reduce

```kotlin
// scan — running aggregate, эмитит каждый промежуточный
flowOf(1, 2, 3).scan(0) { acc, x -> acc + x }
// → 0, 1, 3, 6

// reduce — финальный результат (suspending terminal)
val sum = flowOf(1, 2, 3).reduce { acc, x -> acc + x }  // = 6
```

### flowOn (upstream context)

```kotlin
flow
    .map { heavy(it) }     // выполняется в IO
    .flowOn(Dispatchers.IO)
    .collect { println(it) }  // выполняется в Main
```

`flowOn` меняет context **upstream** (выше по цепочке). Downstream — context collect'а.

### Terminal operators

```kotlin
flow.collect { ... }
flow.collectLatest { ... }   // отменяет предыдущий collect-блок при новом emit
flow.toList()
flow.first()
flow.fold(initial) { acc, x -> ... }
flow.launchIn(scope)         // = scope.launch { collect() }
```

### Frequent patterns

```kotlin
// Auto-search
searchQueryFlow
    .debounce(300)
    .distinctUntilChanged()
    .filter { it.length > 2 }
    .flatMapLatest { query -> repo.search(query) }
    .catch { e -> emit(emptyList()) }
    .collect { results -> _state.value = results }

// Combine multiple sources
combine(userFlow, ordersFlow, promoFlow) { user, orders, promo ->
    HomeUiState(user, orders, promo)
}.stateIn(viewModelScope, WhileSubscribed(5_000), initialState)
```

## Подводные камни

- `flatMapMerge` без `concurrency` лимита → unbounded параллельность, можно перегрузить downstream.
- `flatMapLatest` для критичных операций (запись в БД) → данные могут не дописаться при отмене.
- `catch` НЕ ловит exception в collect-блоке (downstream).
- `flowOn` посредине цепочки — меняет только upstream до этой точки.
- `buffer` без `onBufferOverflow` — `SUSPEND` по умолчанию (producer ждёт).
- `distinctUntilChanged` без кастомного селектора — `equals`, для mutable классов плохо работает.
- `collectLatest` — каждый emit отменяет предыдущий collector; критичная работа может не завершиться.

## Связанные темы

- [[q-09-cold-vs-hot-flow]]
- [[q-13-statein-sharein]]
- [[q-12-callbackflow-channelflow]]

## Follow-up

- Чем `flatMapMerge` отличается от `flatMapConcat`?
- Где использовать `flatMapLatest`?
- Что делают `buffer` и `conflate`?
