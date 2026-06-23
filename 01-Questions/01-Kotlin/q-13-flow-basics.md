---
type: question
category: kotlin
difficulty: senior
tags: [kotlin, coroutines, flow]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Flow: cold vs hot, операторы, `StateFlow` vs `SharedFlow` vs `LiveData`

## Краткий ответ (TL;DR)

`Flow` — холодный асинхронный поток значений, выполняется на каждый collect заново. `StateFlow` — горячий, всегда имеет value, conflate-ит (теряет промежуточные). `SharedFlow` — горячий broadcast, настраиваемый (replay, extra buffer, overflow). `LiveData` — Android-only, lifecycle-aware, проще, но без operator pipeline и менее гибкая.

## Развёрнутый ответ

### Cold Flow

```kotlin
fun numbers(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}
numbers().collect { println(it) }   // запускает производство
numbers().collect { println(it) }   // снова запускает с нуля
```

Производитель выполняется при каждом collect. Никакого состояния не сохраняется. Идеально для запросов с параметрами.

### StateFlow (hot)

```kotlin
private val _state = MutableStateFlow<UiState>(Loading)
val state: StateFlow<UiState> = _state.asStateFlow()
```

- Всегда имеет текущее значение (`value`).
- При collect получаешь сначала текущее, потом обновления.
- **Conflate**: при быстрых обновлениях коллектор получает только последнее.
- **Distinct by default**: если `value == prev` (по equals), коллекторы не получают новое.

Аналог LiveData без lifecycle, но с conflate-семантикой и без primitive-friendly Observer (равенство строгое).

### SharedFlow (hot)

```kotlin
private val _events = MutableSharedFlow<Event>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

- Нет «текущего значения».
- Настраиваемый `replay` (сколько последних эмитов отдавать новым подписчикам).
- Без conflate по умолчанию — буфер.
- Подходит для **one-shot events** (snackbar, navigation), broadcast'ов.

### LiveData

- Lifecycle-aware: `observe(owner, ...)` автоматически отписывается на DESTROYED.
- Conflate, distinct.
- Не пайплайн-friendly (нужны `Transformations`).
- Только main thread для `setValue` (для bg — `postValue`).
- Постепенно вытесняется `StateFlow + repeatOnLifecycle`.

### Сравнение

| | Flow | StateFlow | SharedFlow | LiveData |
|---|------|-----------|------------|----------|
| Cold/Hot | cold | hot | hot | hot |
| Has value | нет | да | нет | да (nullable) |
| Replay | — | 1 | настраивается | 1 |
| Conflate | нет | да | нет (буфер) | да |
| Lifecycle-aware | нет | нет | нет | да |
| Поток | suspend | suspend | suspend | Main |

### Операторы (часто спрашивают)

- `map`, `filter`, `transform` — обычные.
- `flatMapConcat` — последовательно собирает inner flow.
- `flatMapMerge(concurrency)` — параллельно, без порядка.
- `flatMapLatest` — отменяет предыдущий inner при новом эмите (поиск-as-you-type).
- `combine` — комбинирует последние из нескольких flow.
- `zip` — попарно.
- `debounce`, `sample`, `conflate`, `buffer` — backpressure.
- `stateIn(scope, started, initial)` — конвертация в StateFlow.
- `shareIn(scope, started, replay)` — конвертация в SharedFlow.

### Сбор на Android корректно

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }
    }
}
```

`repeatOnLifecycle` отменяет collect при STOP и пере-собирает при START — экономит ресурсы и не получает события в фоне.

## Подводные камни

- `collect` в `lifecycleScope.launch { }` без `repeatOnLifecycle` — продолжает собирать в фоне (Tracker leak).
- `StateFlow` дедуплицирует по `equals` — для mutable state это даёт «не приходит обновление».
- `SharedFlow` с `replay = 0` теряет события без активных подписчиков.
- `flow { }` builder — однопоточный (нельзя emit из других корутин). Для concurrent — `channelFlow { }`.

## Связанные темы

- [[q-10-coroutines-basics]]
- [[q-14-channels-vs-flow]]

## Follow-up

- Чем `flatMapLatest` отличается от `flatMapMerge`?
- Почему `StateFlow` нельзя использовать для one-shot navigation?
- Что произойдёт, если внутри `flow { }` запустить `launch { emit(...) }`?
