---
type: question
category: architecture
difficulty: senior
tags: [mvi, orbit, mvikotlin, store]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# MVI: ручная реализация, Orbit, MVIKotlin, Redux-style

## Краткий ответ (TL;DR)

MVI можно реализовать **руками** (`StateFlow + sealed Intent + reduce`) или через **библиотеки**: **Orbit** (легковесная, intent-based, легко с coroutines), **MVIKotlin** (для KMP, store-based, time-travel debug), **Redux-style** (Mobius от Spotify, Decompose). Главное — single immutable state, intent → reducer → newState, side effects через отдельный канал.

## Развёрнутый ответ

### Ручная реализация

```kotlin
data class CounterState(val value: Int = 0, val loading: Boolean = false)

sealed interface CounterIntent {
    data object Increment : CounterIntent
    data object Decrement : CounterIntent
    data object Reset : CounterIntent
    data object LoadInitial : CounterIntent
}

sealed interface CounterEffect {
    data class Toast(val msg: String) : CounterEffect
}

class CounterVm(private val repo: CounterRepo) : ViewModel() {
    private val _state = MutableStateFlow(CounterState())
    val state: StateFlow<CounterState> = _state.asStateFlow()

    private val _effects = Channel<CounterEffect>()
    val effects = _effects.receiveAsFlow()

    fun onIntent(intent: CounterIntent) {
        when (intent) {
            CounterIntent.Increment -> _state.update { it.copy(value = it.value + 1) }
            CounterIntent.Decrement -> _state.update { it.copy(value = it.value - 1) }
            CounterIntent.Reset -> {
                _state.update { it.copy(value = 0) }
                viewModelScope.launch { _effects.send(CounterEffect.Toast("Reset")) }
            }
            CounterIntent.LoadInitial -> viewModelScope.launch {
                _state.update { it.copy(loading = true) }
                val v = repo.last()
                _state.update { it.copy(value = v, loading = false) }
            }
        }
    }
}
```

### Reducer как функция

Чистый reducer облегчает тестирование:

```kotlin
fun reduce(state: CounterState, action: CounterAction): CounterState = when (action) {
    is CounterAction.SetValue -> state.copy(value = action.value)
    is CounterAction.SetLoading -> state.copy(loading = action.loading)
}

// Test:
@Test fun `reduce SetValue updates value`() {
    val newState = reduce(CounterState(0, false), CounterAction.SetValue(5))
    assertEquals(5, newState.value)
}
```

`Intent` (user-facing) ≠ `Action` (internal). Often:
```
Intent → middleware (async work, calls repo) → emits Actions → reducer → new State
```

### Orbit

```kotlin
class CounterVm(private val repo: CounterRepo) : ContainerHost<CounterState, CounterEffect>, ViewModel() {

    override val container = container<CounterState, CounterEffect>(CounterState())

    fun increment() = intent {
        reduce { state.copy(value = state.value + 1) }
    }

    fun load() = intent {
        reduce { state.copy(loading = true) }
        val v = repo.last()
        reduce { state.copy(value = v, loading = false) }
        postSideEffect(CounterEffect.Toast("Loaded"))
    }
}

// UI:
val state by vm.container.stateFlow.collectAsState()
LaunchedEffect(Unit) {
    vm.container.sideEffectFlow.collect { ... }
}
```

Плюсы Orbit: меньше boilerplate, intent-блоки sequential по умолчанию (нет race), удобный тест-runner.

### MVIKotlin (для KMP)

```kotlin
sealed interface Msg {
    data class ValueChanged(val v: Int) : Msg
}

object CounterReducer : Reducer<State, Msg> {
    override fun State.reduce(msg: Msg): State = when (msg) {
        is Msg.ValueChanged -> copy(value = msg.v)
    }
}

class CounterExecutor : CoroutineExecutor<Intent, Action, State, Msg, Label>() {
    override fun executeIntent(intent: Intent) {
        when (intent) {
            Intent.Increment -> dispatch(Msg.ValueChanged(state().value + 1))
        }
    }
}

val store = StoreFactory.create(
    name = "CounterStore",
    initialState = State(),
    reducer = CounterReducer,
    executorFactory = ::CounterExecutor
)
```

Store-based, разделение на Intent (UI input), Action (system event), Msg (reducer input), Label (one-shot output).

Полезен в Kotlin Multiplatform — shared store между Android/iOS.

### Redux-style паттерны

Принципы:
- **Single store** (часто на весь app).
- **Pure reducer**: `(State, Action) → State`.
- **Middleware** для side effects (analytics, logging, async).
- **DevTools** для time-travel.

В Android реже, потому что app-wide store монолитный → конфликтует с feature-based modularization.

### Тестирование MVI

```kotlin
@Test fun `increment increases value`() = runTest {
    val vm = CounterVm(FakeRepo())
    vm.state.test {
        assertEquals(CounterState(0), awaitItem())
        vm.onIntent(CounterIntent.Increment)
        assertEquals(CounterState(1), awaitItem())
    }
}
```

Reducer тестируется как чистая функция без любых mock'ов.

### Когда MVI не нужен

- Простой экран с 2-3 полями — оверкилл.
- Form с локальным state → проще `var`/`mutableStateOf`.
- Список с pull-to-refresh → MVVM-style с `UiState` достаточно.

Выбор MVI оправдан, когда состояние сложное и/или асинхронных потоков много.

## Подводные камни

- Reducer с side effect (`launch { ... }` внутри) → перестаёт быть pure, time-travel debug не работает.
- Слишком гранулярные Intent (`OnFirstNameKeyPressed`, `OnFirstNameKeyReleased`) → лавина reducer'ов. Группируй по семантике.
- `_state.value = ...` вместо `_state.update {}` → race при concurrent calls (две корутины пишут одновременно).
- Effects через `SharedFlow(replay = 0)` без consumer → теряются. Channel надёжнее.
- Single global store → tight coupling всех фич, conflict при merge.

## Связанные темы

- [[q-01-mvc-mvp-mvvm-mvi]]
- [[q-04-udf]]
- [[q-05-side-effects]]

## Follow-up

- Что такое чистый reducer?
- Чем Orbit удобен по сравнению с ручной реализацией MVI?
- Когда MVI оправдан, а когда нет?
