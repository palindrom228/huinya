---
type: question
category: concurrency
difficulty: senior
tags: [flow, stateflow, sharedflow, livedata]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# SharedFlow vs StateFlow vs LiveData — отличия и когда что

## Краткий ответ (TL;DR)

**StateFlow** = hot, always has value, distinct по equals, аналог BehaviorSubject/LiveData. Для UI state. **SharedFlow** = hot, конфигурируемый (replay, buffer, overflow), без обязательного value. Для events. **LiveData** = legacy, lifecycle-aware из коробки, Main thread only, distinct нет. С Compose — StateFlow + collectAsStateWithLifecycle. LiveData держат для совместимости с Java/старого кода.

## Развёрнутый ответ

### StateFlow

```kotlin
val count: StateFlow<Int> = MutableStateFlow(0)
count.value = 1
count.update { it + 1 }
```

Свойства:
- **Обязательное initial value**.
- `.value` — текущее значение, без suspending.
- **Conflated** — пропускает промежуточные обновления, если subscriber медленный.
- **Distinct** — если новое value == старому (по `equals`), не эмитит.
- Новый subscriber получает **текущее value сразу**.
- Аналог RxJava `BehaviorSubject`.

```kotlin
class CounterVm : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    fun inc() { _count.update { it + 1 } }
}
```

### SharedFlow

```kotlin
val events = MutableSharedFlow<UiEvent>(
    replay = 0,
    extraBufferCapacity = 8,
    onBufferOverflow = BufferOverflow.SUSPEND
)
events.emit(UiEvent.ShowSnackbar("hi"))
```

Параметры:
- **replay** — сколько последних эмишенов получит новый subscriber.
- **extraBufferCapacity** — буфер сверх replay для slow consumers.
- **onBufferOverflow** — `SUSPEND` / `DROP_OLDEST` / `DROP_LATEST`.

Свойства:
- Может не иметь value (replay=0 → новый subscriber ничего не получает до следующего emit).
- НЕ distinct — все эмишены проходят.
- Аналог `PublishSubject` (replay=0) или `ReplaySubject`.

### LiveData

```kotlin
val data = MutableLiveData<String>()
data.value = "hello"
data.postValue("from background")  // безопасно из любого потока

liveData.observe(this) { value -> /* on Main */ }
```

- Lifecycle-aware (auto-unsubscribe на DESTROYED).
- Main thread (set требует Main, postValue безопасный).
- Без backpressure (postValue conflate'ит).
- Без operators (transformations через MediatorLiveData).
- Один initial value опциональный.

### Сравнение

| | StateFlow | SharedFlow | LiveData |
|---|-----------|-----------|----------|
| Initial value | обязателен | опционально (replay) | опционально |
| Distinct | да (equals) | нет | нет |
| Backpressure | conflate | configurable | conflate |
| Subscriber late join | получает current | получает replay (0 default) | получает current |
| Lifecycle aware | через `repeatOnLifecycle` | через `repeatOnLifecycle` | встроено |
| Thread | любой | любой | Main (или postValue) |
| Use case | UI state | events | legacy / Java |

### Use cases

**StateFlow** — для:
- UiState (загрузка, данные, ошибка).
- Form fields (текущий текст).
- Selected item, current screen.

**SharedFlow** — для:
- One-shot events (snackbar, navigation, toast).
- Broadcast событий на много subscribers.
- Replay последних N значений (history).

**LiveData** — для:
- Legacy Java-кода.
- Когда Compose не используется и Activity/Fragment + DataBinding.
- Mediator транспорт (но обычно лучше combine на Flow).

### Conversion

```kotlin
// LiveData → Flow
liveData.asFlow()

// Flow → LiveData (для Java-interop)
flow.asLiveData()

// Flow → StateFlow
flow.stateIn(scope, started, initial)
```

### StateFlow distinct поведение

```kotlin
val s = MutableStateFlow(Item(1, "a"))
s.value = Item(1, "a")   // НЕ эмитится (equals)
s.value = Item(1, "b")   // эмитится

// Тонкость: data class по value equality, обычный class — по reference
```

### One-shot events — почему не StateFlow

```kotlin
// ❌ StateFlow для snackbar
val snackbar: StateFlow<String?> = ...
// проблема: после rotation snackbar показывается снова

// ✅ SharedFlow
val snackbar = MutableSharedFlow<String>()
// или Channel
val snackbar = Channel<String>()
```

См. [[../04-Architecture/q-05-side-effects]].

### MutableStateFlow.update vs value=

```kotlin
// ❌ race condition при concurrent updates
_state.value = _state.value.copy(count = _state.value.count + 1)

// ✅ атомарный CAS
_state.update { it.copy(count = it.count + 1) }
```

`update` — atomic compare-and-set, безопасен от concurrent modification.

### Collect в Compose

```kotlin
@Composable
fun Screen(vm: ScreenVm) {
    // StateFlow → State
    val state by vm.state.collectAsStateWithLifecycle()

    // SharedFlow → effect
    LaunchedEffect(Unit) {
        vm.events.collect { event -> /* handle */ }
    }
}
```

### Collect в Fragment

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { vm.state.collect { render(it) } }
        launch { vm.events.collect { handle(it) } }
    }
}
```

## Подводные камни

- StateFlow distinct по equals → mutable объекты (без data class) не эмитятся повторно.
- `_state.value = ...` без `update` → race condition при concurrent.
- SharedFlow `replay > 0` + one-shot events → новый subscriber получит старое событие.
- LiveData `setValue` из background → IllegalStateException; нужен `postValue`.
- Подписка на StateFlow без `repeatOnLifecycle` в Fragment → утечки/обновления в background.
- `tryEmit` на SharedFlow с `extraBufferCapacity=0` и subscribers — false (буфер полон).

## Связанные темы

- [[q-09-cold-vs-hot-flow]]
- [[q-13-statein-sharein]]
- [[../04-Architecture/q-05-side-effects]]

## Follow-up

- Чем StateFlow отличается от SharedFlow?
- Почему StateFlow не подходит для one-shot events?
- Зачем `update { }` вместо `value = `?
