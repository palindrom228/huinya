---
type: question
category: concurrency
difficulty: senior
tags: [flow, stateIn, shareIn, sharingstarted]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# stateIn / shareIn и SharingStarted стратегии

## Краткий ответ (TL;DR)

`stateIn` превращает cold Flow в StateFlow (1 значение, distinct). `shareIn` — в SharedFlow (configurable replay). Оба нужны чтобы (1) **не перезапускать** cold flow на каждый subscribe, (2) **расшарить** upstream между subscribers. `SharingStarted` контролирует когда upstream активен: `Eagerly` (сразу), `Lazily` (по первому subscriber), `WhileSubscribed(stopTimeout, replayExpiration)` (актив пока есть subscribers + grace period).

## Развёрнутый ответ

### Проблема: cold flow в нескольких subscribers

```kotlin
val news: Flow<List<News>> = repo.observeNews()  // cold — каждый collect = новый DB query

// В UI 3 collector'а → 3 запроса
launch { news.collect { /* feed */ } }
launch { news.collect { /* widget */ } }
launch { news.collect { /* notification */ } }
```

С `shareIn`:

```kotlin
val news = repo.observeNews()
    .shareIn(scope, SharingStarted.WhileSubscribed(5_000), replay = 1)

// 3 collector'а → 1 DB query, шарят результат
```

### stateIn

```kotlin
val uiState: StateFlow<UiState> = repo.observeData()
    .map { UiState.Success(it) }
    .catch { emit(UiState.Error(it.message)) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = UiState.Loading
    )
```

- Возвращает `StateFlow<T>` (всегда есть value).
- `initialValue` — обязательный.
- Distinct по equals.

### shareIn

```kotlin
val events: SharedFlow<Event> = source
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.Lazily,
        replay = 0
    )
```

- Возвращает `SharedFlow<T>` (может быть без value).
- `replay` — сколько последних эмишенов получит новый subscriber.

### SharingStarted стратегии

#### Eagerly

```kotlin
.stateIn(scope, SharingStarted.Eagerly, initial)
```

Upstream активен с момента создания и до cancel scope. Работает даже без subscribers. **Не рекомендуется** для тяжёлых flow.

#### Lazily

```kotlin
.stateIn(scope, SharingStarted.Lazily, initial)
```

Стартует на первом subscriber, работает до cancel scope (не останавливается). Подходит для one-time данных.

#### WhileSubscribed

```kotlin
.stateIn(
    scope,
    SharingStarted.WhileSubscribed(
        stopTimeoutMillis = 5_000,
        replayExpirationMillis = Long.MAX_VALUE
    ),
    initial
)
```

- Стартует на первом subscriber.
- При нуле subscribers ждёт `stopTimeoutMillis` (grace period для rotation).
- Если за это время появился subscriber — продолжает работать.
- Иначе останавливает upstream.
- `replayExpirationMillis` — через сколько после остановки сбрасывается cached value.

**Рекомендация Google**: `WhileSubscribed(5_000)` для UI flows.

### Почему 5 секунд

Configuration change (rotation) → Activity destroy → recreate → новый subscribe. Если бы upstream останавливался моментально, между этими событиями flow пересоздавался бы, теряя cache. 5 сек — достаточно для rotation.

### Eagerly опасность

```kotlin
class HomeVm(repo: NewsRepository) : ViewModel() {
    // ❌ Eagerly — даже если экран никогда не открыт, упорно тянет данные
    val news = repo.observeNews()
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())
}
```

VM создаётся при первом доступе → Eagerly запускает upstream → если VM создан "на всякий случай", тратит ресурсы.

### Comparison

| | Eagerly | Lazily | WhileSubscribed(5_000) |
|--|---------|--------|------------------------|
| Старт | сразу при создании | первый subscriber | первый subscriber |
| Стоп | cancel scope | cancel scope | через 5 сек после последнего unsubscribe |
| Использование | критичные данные | one-time данные | UI flows (Compose, Fragment) |
| Энергоэффективность | плохо | средне | хорошо |

### Custom SharingStarted

```kotlin
object WhileVisible : SharingStarted {
    override fun command(subscriptionCount: StateFlow<Int>) =
        subscriptionCount.map { if (it > 0) SharingCommand.START else SharingCommand.STOP }
}
```

Редко нужно. Стандартные стратегии покрывают 95% случаев.

### Pattern: combine + stateIn

```kotlin
val uiState: StateFlow<HomeUiState> = combine(
    repo.observeUser(),
    repo.observeOrders(),
    repo.observePromo()
) { user, orders, promo ->
    HomeUiState(user, orders, promo)
}
    .stateIn(viewModelScope, WhileSubscribed(5_000), HomeUiState.Loading)
```

### shareIn для events

```kotlin
val analyticsEvents = userActions
    .shareIn(scope, SharingStarted.Eagerly, replay = 0)
```

`Eagerly` тут логично: события надо собирать всегда, даже без UI subscribers.

### onSubscription

```kotlin
flow
    .onSubscription { emit(Loading) }   // на каждом subscribe
    .shareIn(scope, WhileSubscribed(5_000))
```

### subscriptionCount

```kotlin
val sharedFlow = source.shareIn(scope, WhileSubscribed(5_000))
sharedFlow.subscriptionCount.collect { count ->
    println("now $count subscribers")
}
```

Полезно для debugging/metrics.

## Подводные камни

- `Eagerly` для тяжёлого flow + VM создан "впрок" → расходы зря.
- `WhileSubscribed(0)` (timeout=0) → между rotation flow пересоздаётся, теряет cache.
- `stateIn` с `Lazily` — после первого subscriber upstream работает до cancel scope (нет stop при unsubscribe).
- `replay = 0` для StateFlow невозможен — у него всегда есть value.
- Создал `stateIn` в Composable scope (rememberCoroutineScope) → каждый recomposition пересоздаёт.
- `shareIn(scope, Eagerly, replay = 0)` — субscriber, пришедший после первого emit, ничего не увидит.

## Связанные темы

- [[q-09-cold-vs-hot-flow]]
- [[q-10-sharedflow-stateflow-livedata]]
- [[q-11-flow-operators]]

## Follow-up

- Зачем `stateIn` / `shareIn`?
- Что делает `WhileSubscribed(5_000)` и почему именно 5 сек?
- Когда `Eagerly`, когда `Lazily`, когда `WhileSubscribed`?
