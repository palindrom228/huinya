---
type: question
category: architecture
difficulty: senior
tags: [side-effects, events, navigation]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# Side effects: one-shot events, navigation, snackbar — как отделить от state

## Краткий ответ (TL;DR)

**State** — то, что UI рисует постоянно (отображается всегда, восстанавливается при recreation). **Event** (side effect) — одноразовое действие: navigate, snackbar, toast. Хранить как state нельзя — после rotation сработает снова. Решения: **Channel** (consumed once), **SharedFlow** с правильным реплеем, или **state-with-handled-flag** (например, `Event<T>` с `getContentIfNotHandled`).

## Развёрнутый ответ

### Проблема

```kotlin
// ❌ snackbar как state
data class State(val message: String?)

vm.state.collect { state ->
    if (state.message != null) showSnackbar(state.message)
}
// После rotation message всё ещё в state → snackbar повторно
```

### Решение 1: Channel

```kotlin
class MyVm : ViewModel() {
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun submit() = viewModelScope.launch {
        runCatching { repo.submit() }
            .onSuccess { _events.send(UiEvent.NavigateBack) }
            .onFailure { _events.send(UiEvent.ShowError(it.message ?: "")) }
    }
}

sealed interface UiEvent {
    data object NavigateBack : UiEvent
    data class ShowError(val msg: String) : UiEvent
}

// Compose:
LaunchedEffect(Unit) {
    vm.events.collect { event ->
        when (event) {
            UiEvent.NavigateBack -> navController.popBackStack()
            is UiEvent.ShowError -> snackbarHostState.showSnackbar(event.msg)
        }
    }
}
```

`Channel` — каждое сообщение получает **один** consumer. Если consumer не collect'ит — буфер.

### Проблема Channel в Compose

`LaunchedEffect(Unit)` отменяется при выходе с экрана → если event отправили в этот момент, он буферизуется в Channel, при возврате consume'ится. Норм для navigation/snackbar.

Но: при rotation `LaunchedEffect` отменяется и пересоздаётся → события в буфере прочитаются повторно. Channel.BUFFERED + новая подписка → ok (один раз), но **если consumer вышел до collect — событие пропадёт**. Использовать lifecycle-aware:

```kotlin
val lifecycleOwner = LocalLifecycleOwner.current
LaunchedEffect(lifecycleOwner) {
    lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        vm.events.collect { ... }
    }
}
```

### Решение 2: SharedFlow с buffer

```kotlin
private val _events = MutableSharedFlow<UiEvent>(extraBufferCapacity = 1)
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

fun submit() = viewModelScope.launch {
    _events.emit(UiEvent.NavigateBack)
}
```

Минус: SharedFlow `replay = 0` без collectors при emit — событие теряется. Для navigation ok (UI должна быть на экране).

### Решение 3: Event wrapper (legacy)

```kotlin
class Event<out T>(private val content: T) {
    var hasBeenHandled = false; private set
    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) null else { hasBeenHandled = true; content }
    }
}

val showSnackbar = MutableLiveData<Event<String>>()

// observer:
vm.showSnackbar.observe(viewLifecycleOwner) { event ->
    event.getContentIfNotHandled()?.let { showSnackbar(it) }
}
```

Использовалось с LiveData. С `StateFlow` — устарело, лучше Channel/SharedFlow.

### Navigation как side effect

```kotlin
sealed interface NavCommand {
    data object Back : NavCommand
    data class To(val route: String) : NavCommand
}

private val _nav = Channel<NavCommand>()
val nav = _nav.receiveAsFlow()

// в Composable:
LaunchedEffect(Unit) {
    vm.nav.collect { cmd ->
        when (cmd) {
            NavCommand.Back -> navController.popBackStack()
            is NavCommand.To -> navController.navigate(cmd.route)
        }
    }
}
```

Альтернатива: passing `onNavigateBack: () -> Unit` lambda из caller. Проще, но меньше контроля у VM.

### Snackbar в Material 3 Compose

```kotlin
val snackbarHostState = remember { SnackbarHostState() }

LaunchedEffect(Unit) {
    vm.events.collect { event ->
        if (event is UiEvent.ShowSnackbar) {
            snackbarHostState.showSnackbar(event.msg)
        }
    }
}

Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { ... }
```

### Что считать state vs event

| Признак | State | Event |
|---------|-------|-------|
| Отображается всегда | да | нет |
| После rotation важно сохранить | да | обычно нет (или одноразово) |
| UI = функция от него | да | нет (триггерит действие) |
| Пример | `isLoading`, `user`, `items` | `navigateBack`, `showSnackbar`, `toast` |

### Best practice (Google guidance)

[Google guide](https://developer.android.com/topic/architecture/ui-layer/events) рекомендует:
- State → `StateFlow<UiState>`.
- One-shot events с лимитом: предпочесть **выразить через state**, если возможно (например, `errorMessage: String?` который UI сбрасывает после показа через `vm.onErrorShown()`).

```kotlin
data class State(val items: List<X>, val error: String? = null)

// UI:
state.error?.let {
    LaunchedEffect(it) {
        snackbarHostState.showSnackbar(it)
        vm.onErrorShown()
    }
}

// VM:
fun onErrorShown() { _state.update { it.copy(error = null) } }
```

## Подводные камни

- SharedFlow без `repeatOnLifecycle` — события теряются если UI в background.
- Channel.UNLIMITED + забыли collect → утечка памяти (буфер растёт).
- Toast в ViewModel напрямую (`Toast.makeText(context, ...)`) — VM получила context → не testable, лик.
- Snackbar как `Boolean` state — после rotation покажется снова.
- Navigation lambda в Composable, вызвана из background coroutine → NavController not on main thread.

## Связанные темы

- [[q-04-udf]]
- [[../05-Concurrency/q-shared-flow-state-flow]]
- [[../03-UI/q-03-compose-side-effects]]

## Follow-up

- Почему `showSnackbar: Boolean` как state плохо?
- Чем `Channel` отличается от `SharedFlow(replay = 0)` для событий?
- Как `Event<T>` wrapper решал проблему до Channel/Flow?
