---
type: question
category: concurrency
difficulty: senior
tags: [rxjava, coroutines, flow, migration]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# RxJava vs Coroutines/Flow: миграция и сравнение

## Краткий ответ (TL;DR)

**RxJava** — reactive streams, push-based, операторы (map/filter/flatMap), Schedulers, Subjects. **Coroutines + Flow** — structured concurrency, suspend functions, cold Flow / hot StateFlow / SharedFlow. Современный Android **рекомендует корутины**: проще читать, lifecycle-aware, меньше boilerplate, без `Disposable` ручного управления. Миграция: `rxjava3-coroutines` bridge (`.asFlow()`, `.await()`), постепенный переход по слоям (сначала repository, потом VM). Rx ещё нужен для legacy кода и сложных combinator-цепочек.

## Развёрнутый ответ

### Концептуально

| | RxJava | Coroutines + Flow |
|---|--------|-------------------|
| Парадигма | Reactive streams | Structured concurrency + Flow |
| Async unit | `Single`, `Observable`, `Flowable`, `Maybe`, `Completable` | `suspend fun`, `Flow<T>`, `StateFlow`, `SharedFlow` |
| Cancellation | `Disposable.dispose()` ручная | автоматический (scope cancel) |
| Backpressure | `Flowable`, стратегии | suspend = natural backpressure |
| Lifecycle | RxLifecycle / AutoDispose | viewModelScope / lifecycleScope |
| Threading | `subscribeOn`/`observeOn` + Schedulers | Dispatchers + withContext |
| Чтение кода | цепочки операторов | sequential suspend код |
| Compile-time safety | runtime errors в onError | suspend в типе, обычный try/catch |

### Простой пример

```kotlin
// RxJava
api.getUser(id)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
        { user -> ui.show(user) },
        { error -> ui.error(error) }
    )

// Coroutines
viewModelScope.launch {
    try {
        val user = withContext(Dispatchers.IO) { api.getUser(id) }
        ui.show(user)
    } catch (e: Exception) {
        ui.error(e)
    }
}
```

### Соответствие типов

| RxJava | Kotlin coroutines |
|--------|-------------------|
| `Single<T>` | `suspend fun(): T` |
| `Maybe<T>` | `suspend fun(): T?` |
| `Completable` | `suspend fun(): Unit` |
| `Observable<T>` / `Flowable<T>` | `Flow<T>` |
| `Subject` / `BehaviorSubject` | `SharedFlow` / `StateFlow` |
| `PublishSubject` | `SharedFlow(replay = 0)` |
| `BehaviorSubject` | `StateFlow` (всегда current value) |
| `ReplaySubject` | `SharedFlow(replay = N)` |
| `Disposable` | `Job` |
| `CompositeDisposable` | scope (`coroutineScope`, `viewModelScope`) |

### Операторы

| RxJava | Flow |
|--------|------|
| `map` | `map` |
| `filter` | `filter` |
| `flatMap` | `flatMapConcat` / `flatMapMerge` |
| `switchMap` | `flatMapLatest` |
| `concatMap` | `flatMapConcat` |
| `zip` | `zip` |
| `combineLatest` | `combine` |
| `merge` | `merge` |
| `debounce` | `debounce` |
| `distinctUntilChanged` | `distinctUntilChanged` |
| `take` | `take` |
| `scan` | `scan` |
| `startWith` | `onStart { emit(...) }` |
| `onErrorReturn` | `catch { emit(...) }` |
| `retry` | `retry` |
| `doOnNext` | `onEach` |

### Schedulers → Dispatchers

| RxJava Scheduler | Coroutines Dispatcher |
|------------------|-----------------------|
| `Schedulers.io()` | `Dispatchers.IO` |
| `Schedulers.computation()` | `Dispatchers.Default` |
| `AndroidSchedulers.mainThread()` | `Dispatchers.Main` |
| `Schedulers.single()` | `newSingleThreadContext()` |
| `Schedulers.from(executor)` | `executor.asCoroutineDispatcher()` |

### Hot vs Cold

- RxJava: `Observable.create` — cold. `Subject` — hot.
- Coroutines: `flow { }` — cold. `StateFlow` / `SharedFlow` — hot.

### Backpressure

- RxJava: `Flowable` с стратегиями (`BUFFER`, `DROP`, `LATEST`, `ERROR`).
- Coroutines: `Flow` natural backpressure через `suspend emit`. `buffer()`, `conflate()`, `collectLatest`.

```kotlin
flow.buffer(capacity = 64)
flow.conflate()                // keep latest
flow.collectLatest { ... }     // cancel previous on new
```

### Subjects → SharedFlow/StateFlow

```kotlin
// RxJava BehaviorSubject
val subject = BehaviorSubject.createDefault(0)
subject.onNext(1)
subject.subscribe { println(it) }

// Coroutines StateFlow
val state = MutableStateFlow(0)
state.value = 1
state.collect { println(it) }
```

### Bridges (rxjava3-coroutines)

```kotlin
// RxJava → coroutines
val user: User = api.getUserRx(id).await()       // Single → suspend
val flow: Flow<User> = obs.asFlow()              // Observable → Flow

// coroutines → RxJava
val single: Single<User> = rxSingle { fetchUser() }
val obs: Observable<Int> = flow.asObservable()
```

Артефакт: `kotlinx-coroutines-rx3`.

### Стратегия миграции

1. **Добавь корутины параллельно**. Существующий Rx-код не трогай.
2. **Низ стека первым**: API/DAO — `suspend fun` вместо `Single`. Bridge для остального.
3. **Repository слой** — `Flow<T>` вместо `Observable<T>`.
4. **ViewModel** — `viewModelScope.launch`, `StateFlow`.
5. **UI** — `collectAsStateWithLifecycle()` / `lifecycleScope`.
6. **Удали `CompositeDisposable`**, замена — scope.

### Lifecycle integration

```kotlin
// RxJava — нужен RxLifecycle / AutoDispose
disposable.add(
    api.getUser().subscribe(...)
)
override fun onDestroy() { disposable.clear() }

// Coroutines — автоматически
viewModelScope.launch { api.getUser() }
// отменится в onCleared()
```

### Error handling

```kotlin
// RxJava
single
    .onErrorReturn { fallback }
    .doOnError { log(it) }
    .subscribe(...)

// Coroutines
try {
    val r = call()
} catch (e: IOException) {
    fallback
}

// Flow
flow
    .catch { e -> emit(fallback) }
    .onEach { ... }
    .collect()
```

### Тестирование

```kotlin
// RxJava — TestObserver
api.getUser(1).test()
    .assertValue(expectedUser)
    .assertComplete()

// Coroutines — runTest + Turbine
@Test fun test() = runTest {
    repo.observeUsers().test {
        assertEquals(user1, awaitItem())
        awaitComplete()
    }
}
```

### Что Rx делает лучше

- Сложные **declarative pipelines** с многими операторами — Rx читается компактнее.
- Зрелые операторы (например, `window`, `groupBy`).
- Backpressure стратегии явные.
- Legacy экосистема (RxBinding, RxPermissions).

### Что корутины делают лучше

- **Sequential код** — try/catch, if, for — обычный Kotlin.
- **Structured concurrency** — нет утечек по умолчанию.
- **Lifecycle integration** — viewModelScope из коробки.
- **Меньше overhead** — coroutine ≪ thread, в отличие от Rx subscriptions с allocations.
- **Compose / KMP** — first-class support.

### Compose interop

- `flow.collectAsStateWithLifecycle()` — natural.
- Rx → Compose: `observable.subscribeAsState()` (rxjava3-compose), но менее естественно.

### Hybrid: что хранить на Rx

Если кодовая база большая и Rx-код стабилен — оставь legacy. Новые фичи — корутины. Bridge только на границах.

### Performance

- Корутина: ~200 байт overhead, lazy.
- Rx Observable: больше allocations на subscription/operator.
- В hot path — корутины обычно дешевле.

## Подводные камни

- Микс Rx + coroutines без чёткой стратегии → confusion, дублирующиеся подписки.
- Забытый `dispose` → memory leak. Перейди на AutoDispose / scope.
- `Subject` без replay→ потеря initial value. В Coroutines — StateFlow.
- `Schedulers.io()` vs `Dispatchers.IO` — разные тред-пулы, не смешивай в одной операции.
- `Flow` cold ≠ Rx hot — частая ошибка миграции, нужен `shareIn`/`stateIn`.
- `Observable.error()` без `.onErrorReturn` → crash UI subscriber. Coroutines — try/catch / catch.
- `combineLatest` с большим числом источников в Rx → race. В Flow `combine` — атомарный snapshot.

## Связанные темы

- [[q-04-cold-vs-hot-flow]]
- [[q-17-threads-executors]]
- [[q-07-exception-handling]]

## Follow-up

- Зачем мигрировать с Rx на корутины?
- Чем `Flow` отличается от `Observable`?
- Что делает `flatMapLatest`, какой Rx-эквивалент?
