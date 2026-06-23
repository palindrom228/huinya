---
type: question
category: architecture
difficulty: middle
tags: [mvvm, mvi, mvp, presentation-pattern]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# MVC / MVP / MVVM / MVI — отличия, когда что

## Краткий ответ (TL;DR)

**MVC** — Controller обновляет View и Model; на Android чаще плохо разделено. **MVP** — Presenter держит ссылку на View interface, явно вызывает render-методы. **MVVM** — ViewModel экспонирует **observable state**, View подписывается; нет ссылки на View. **MVI** — `(State, Intent) → State` через reducer, **immutable state**, single source. Современный стандарт Android = MVVM-style с unidirectional data flow (часто фактически MVI).

## Развёрнутый ответ

### MVC

```
View ⇄ Controller ⇄ Model
```

Контроллер обрабатывает input, мутирует Model, обновляет View. На Android: Activity = и View, и Controller → smell. Чистого MVC почти нет.

### MVP

```
View interface ← Presenter → Model
```

```kotlin
interface UserView {
    fun showLoading()
    fun showUser(user: User)
    fun showError(message: String)
}

class UserPresenter(private val view: UserView, private val repo: UserRepo) {
    fun loadUser(id: Long) {
        view.showLoading()
        scope.launch {
            runCatching { repo.get(id) }
                .onSuccess(view::showUser)
                .onFailure { view.showError(it.message ?: "") }
        }
    }
}
```

Плюсы: testable (мокаем View interface). Минусы: presenter держит View → лик при rotation, нужно явно `attachView/detachView`.

### MVVM

```
View → подписка → ViewModel → Repository
       ←  state  ←
```

ViewModel **не знает о View**. View наблюдает `StateFlow`/`LiveData`/`State`.

```kotlin
class UserVm(private val repo: UserRepo) : ViewModel() {
    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state: StateFlow<UiState> = _state.asStateFlow()

    fun load(id: Long) = viewModelScope.launch {
        _state.value = UiState.Loading
        runCatching { repo.get(id) }
            .onSuccess { _state.value = UiState.Data(it) }
            .onFailure { _state.value = UiState.Error(it.message) }
    }
}

// Compose:
val state by vm.state.collectAsStateWithLifecycle()
```

Плюсы: rotation-safe (ViewModel переживёт), testable без mock View, легко composable.

### MVI

`Model-View-Intent` — формализация unidirectional flow:

```kotlin
data class State(val isLoading: Boolean, val user: User?, val error: String?)

sealed interface Intent {
    data class Load(val id: Long) : Intent
    data object Retry : Intent
}

class UserVm : ViewModel() {
    private val _state = MutableStateFlow(State(true, null, null))
    val state = _state.asStateFlow()

    fun onIntent(intent: Intent) {
        when (intent) {
            is Intent.Load -> load(intent.id)
            is Intent.Retry -> load(lastId)
        }
    }

    private fun reduce(current: State, event: Event): State = when (event) {
        is Event.Loaded -> current.copy(isLoading = false, user = event.user)
        is Event.Failed -> current.copy(isLoading = false, error = event.msg)
    }
}
```

Ключевые принципы:
- **Single immutable State** — UI = функция от State.
- **Intent** (или Action/Event) — единственный способ изменить состояние.
- **Reducer** — чистая функция `(State, Event) → State`.
- **Side effects** (one-shot: navigation, snackbar) — отдельный канал (Channel/SharedFlow).

Плюсы: предсказуемость, time-travel debugging, легко тестировать reducer. Минусы: больше boilerplate, для простых экранов overkill.

### Какой выбрать

- **Маленькое приложение / прототип** — MVVM с `StateFlow<UiState>`.
- **Сложный экран с многими интент'ами** — MVI (формализованный).
- **Legacy Java + RxJava** — MVP может ещё жить.
- **Никогда** — чистый MVC на Android.

Современный Android (Google guide) — MVVM по сути, но shape близко к MVI (sealed UiState, events через Flow).

### Sealed UiState

```kotlin
sealed interface ProfileUiState {
    data object Loading : ProfileUiState
    data class Success(val user: User) : ProfileUiState
    data class Error(val message: String) : ProfileUiState
}
```

Лучше, чем `data class State(isLoading, user, error)` — компилятор enforce'ит exhaustive when, нет невалидных комбинаций.

## Подводные камни

- MVVM с публичной mutable state (`var counter: Int`) — не MVVM. Использовать `StateFlow`.
- MVI без отдельного канала для one-shot events → snackbar показывается после rotation повторно.
- Presenter в MVP, который пережил View (rotation) → NPE при `view.showX()`. Решение: lifecycle-aware presenter.
- "MVVM" с `getViewModel()` который держит `context.getActivity()` — лик, не MVVM.
- Reducer с side effects (network call) → перестаёт быть pure, ломает testability.

## Связанные темы

- [[q-02-viewmodel]]
- [[q-04-udf]]
- [[q-05-side-effects]]

## Follow-up

- Чем MVVM отличается от MVP в плане owner reference?
- Что такое reducer в MVI?
- Почему single immutable State лучше нескольких MutableLiveData?
