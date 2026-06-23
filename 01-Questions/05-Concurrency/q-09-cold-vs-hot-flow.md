---
type: question
category: concurrency
difficulty: middle
tags: [flow, cold, hot, builders]
created: 2026-06-16
last_reviewed: 2026-06-16
times_failed: 0
times_passed: 1
status: learning
---

# Cold Flow vs Hot Flow, builders

## Краткий ответ (TL;DR)

**Cold Flow** — упражнения по коду: каждый `collect` запускает producer заново, у каждого subscriber свой поток данных. Builders: `flow { }`, `flowOf`, `asFlow`. **Hot Flow** — producer уже работает, subscribers получают то, что есть/будет: `StateFlow`, `SharedFlow`, `Channel.receiveAsFlow`. `callbackFlow` / `channelFlow` — cold по контракту (запускают на collect), но внутри hot-like.

## Развёрнутый ответ

### Cold Flow

```kotlin
val numbers: Flow<Int> = flow {
    println("producing")
    emit(1); emit(2); emit(3)
}

numbers.collect { println("A: $it") }   // печатает "producing" + A:1,2,3
numbers.collect { println("B: $it") }   // печатает "producing" снова + B:1,2,3
```

Каждый `collect` = новый запуск flow builder'а.

### Hot Flow (StateFlow / SharedFlow)

```kotlin
val state = MutableStateFlow(0)
launch { state.collect { println("A: $it") } }
state.value = 1
launch { state.collect { println("B: $it") } }
state.value = 2
// A: 0, A: 1, B: 1, A: 2, B: 2
```

`StateFlow` — value существует независимо от subscribers; новый subscriber получает текущее значение.

### Cold builders

#### `flow { }`

```kotlin
val ticker: Flow<Int> = flow {
    var i = 0
    while (true) {
        emit(i++)
        delay(1000)
    }
}
```

Suspend builder, `emit` — suspending. Каждый collect стартует с i=0.

#### `flowOf`

```kotlin
val three = flowOf(1, 2, 3)   // эмитит 1,2,3 и завершается
```

#### `asFlow`

```kotlin
listOf(1, 2, 3).asFlow()
(1..100).asFlow()
arrayOf("a", "b").asFlow()
```

#### `callbackFlow`

```kotlin
fun locationFlow(): Flow<Location> = callbackFlow {
    val callback = object : LocationCallback() {
        override fun onResult(loc: Location) {
            trySend(loc)   // не suspending
        }
    }
    locationClient.requestUpdates(callback)
    awaitClose { locationClient.removeUpdates(callback) }
}
```

- Создаёт Channel внутри.
- `awaitClose` — обязательно для cleanup (отписаться от callback при collect cancel).
- Cold: collect → register listener, cancel → unregister.

#### `channelFlow`

```kotlin
fun merged(): Flow<Item> = channelFlow {
    launch { source1.collect { send(it) } }
    launch { source2.collect { send(it) } }
}
```

Для случаев, когда из flow нужно `emit` из других корутин/dispatcher'ов (обычный `flow { }` строго sequential, нельзя `emit` из background launch).

### Hot builders / conversions

#### MutableStateFlow

```kotlin
val count = MutableStateFlow(0)
count.value = 1
count.update { it + 1 }   // атомарный CAS
```

#### MutableSharedFlow

```kotlin
val events = MutableSharedFlow<UiEvent>(
    replay = 0,
    extraBufferCapacity = 16,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
events.tryEmit(UiEvent.ShowSnackbar("hi"))
```

#### stateIn / shareIn — cold → hot

```kotlin
val items: StateFlow<List<Item>> = repo.observeItems()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = emptyList()
    )
```

См. [[q-13-statein-sharein]].

### Cold vs Hot — сравнение

| | Cold | Hot |
|--|------|-----|
| Producer запускается | на каждый collect | один раз |
| Subscribers видят | свой полный поток | только то, что происходит сейчас |
| Без subscribers | нет работы | продолжает работать (StateFlow value существует) |
| Использование | данные по запросу (DB query, API call) | события, состояние |

### Examples

```kotlin
// Cold: каждый collect = новый DB query
fun observeUser(id: Long): Flow<User> = flow {
    val user = db.getUser(id)
    emit(user)
}

// Hot: state, одно значение для всех
val currentUser = MutableStateFlow<User?>(null)
```

### emit vs tryEmit

- `emit` — suspending, ждёт места в буфере.
- `tryEmit` — non-suspending, возвращает Boolean (false если буфер полон).
- `send` (Channel) — suspending.
- `trySend` (Channel/callbackFlow) — non-suspending.

### Flow.cancellable

Flow по умолчанию проверяет cancellation на каждом `emit`. Если внутри tight loop без emit:

```kotlin
flow {
    repeat(1_000_000) {
        if (it % 1000 == 0) emit(it)  // checkpoint
    }
}

// или явно:
flow { ... }.cancellable()
```

### Cold flow и lifecycle

```kotlin
// ❌ в Compose
LaunchedEffect(Unit) {
    repo.observeUser().collect { /* ... */ }
}

// ✅ преобразовать в hot
val user by repo.observeUser()
    .collectAsStateWithLifecycle(initialValue = null)
```

`collectAsStateWithLifecycle` — учитывает Activity lifecycle, не collect в background.

## Подводные камни

- `flow { }` нельзя emit из другой корутины (строгий context check) → используй `channelFlow`.
- `callbackFlow` без `awaitClose` → не отпишется от callback, leak.
- `MutableSharedFlow` без `extraBufferCapacity` и без `replay` — `tryEmit` всегда возвращает false для suspending subscribers.
- StateFlow — distinct по `equals` (повторное `value = x` тем же значением не эмитит). Не подходит для "событий".
- Cold flow подписка из ViewModel напрямую → перезапускается при rotation, если не `stateIn`.

## Связанные темы

- [[q-10-sharedflow-stateflow-livedata]]
- [[q-12-callbackflow-channelflow]]
- [[q-13-statein-sharein]]

## Follow-up

- Чем cold flow отличается от hot?
- Когда `callbackFlow`, когда `channelFlow`?
- Почему `MutableSharedFlow.tryEmit` может вернуть false?
